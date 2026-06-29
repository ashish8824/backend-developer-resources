# Day 4 — DTOs, Validation & Serialization
### NestJS Interview Prep | 3-Year Backend Engineer Level

> **Goal for today:** Master how data flows IN and OUT of your NestJS API safely. Understand DTOs (what they are and why they exist), every important class-validator decorator, class-transformer for type conversion, response serialization to hide sensitive fields, and whitelisting strategies. A 3-year engineer is expected to know ALL of this deeply.

---

## 🧭 First-Timer Guidance — Read This First

**The core problem this day solves:**

Your API receives raw JSON from the internet. That JSON could contain:
- Missing required fields
- Wrong data types (`"age": "twenty"` instead of `"age": 20`)
- Extra dangerous fields (`"isAdmin": true`)
- Malformed values (`"email": "notanemail"`)
- SQL injection attempts in string fields

And your API sends JSON back to clients. That response might accidentally include:
- User passwords (even hashed)
- Internal database IDs for sensitive tables
- Security tokens
- Fields only admins should see

**DTOs and Validation solve the INPUT problem.**
**Serialization solves the OUTPUT problem.**

**Mental model for today:**

```
Client sends JSON
      ↓
   [DTO + Validation]  ← class-validator: "Is this data valid?"
      ↓                  class-transformer: "Convert types"
   Controller/Service   ← whitelist: "Strip unknown fields"
      ↓
   [Serialization]     ← class-transformer: "Hide sensitive fields"
      ↓
Client receives JSON
```

**Setup — install these packages first:**
```bash
npm install class-validator class-transformer
```

These two packages work TOGETHER. `class-validator` adds validation rules. `class-transformer` converts plain objects to/from class instances (required for validation to work) and handles field exclusion.

---

## Table of Contents

1. [What is a DTO and Why Does It Exist?](#1-what-is-a-dto-and-why-does-it-exist)
2. [class-validator — Every Important Decorator](#2-class-validator--every-important-decorator)
3. [ValidationPipe — Deep Configuration](#3-validationpipe--deep-configuration)
4. [Nested Object Validation](#4-nested-object-validation)
5. [class-transformer — Type Conversion](#5-class-transformer--type-conversion)
6. [Response Serialization — Hiding Sensitive Fields](#6-response-serialization--hiding-sensitive-fields)
7. [Whitelisting & Security Strategies](#7-whitelisting--security-strategies)
8. [Mapped Types — DRY DTOs](#8-mapped-types--dry-dtos)
9. [Custom Validation Decorators](#9-custom-validation-decorators)
10. [Real-World DTO Patterns](#10-real-world-dto-patterns)
11. [Interview Questions & Answers](#11-interview-questions--answers)

---

## 1. What is a DTO and Why Does It Exist?

### What is a DTO?
**DTO = Data Transfer Object.** It is a class that defines the **shape of data** coming INTO (or going out of) your API.

### Simple explanation
A DTO is a blueprint that says: "If you want to create a user, you MUST send these fields, in these formats, with these constraints." It's a contract between your API and the people who call it.

### Technical explanation

Without DTOs, you'd have this:
```typescript
// ❌ Without DTO — dangerous and unmaintainable
@Post()
async create(@Body() body: any) {
  // body is type 'any' — TypeScript gives no help
  // You don't know what fields exist
  // No validation — client could send anything
  // You might accidentally save extra fields to DB
  
  return this.usersService.create(body);
}
```

With DTOs, you get this:
```typescript
// ✅ With DTO — safe, documented, and validated
@Post()
async create(@Body() createUserDto: CreateUserDto) {
  // TypeScript knows EXACTLY what fields exist
  // ValidationPipe automatically validates before this method runs
  // Unknown fields are stripped by whitelist
  // This method only runs if data is 100% valid
  
  return this.usersService.create(createUserDto);
}
```

### Why DTOs are separate classes (not interfaces)

```typescript
// ❌ Interface — TypeScript only, erased at runtime
interface CreateUserDto {
  name: string;
  email: string;
}
// At runtime (JavaScript), interfaces don't exist
// class-validator CANNOT validate interfaces
// ValidationPipe cannot work with interfaces

// ✅ Class — exists at runtime as real JavaScript
export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsEmail()
  email: string;
}
// At runtime, class-validator can inspect this class
// and run validation based on the decorators
// This is WHY NestJS uses classes for DTOs, not interfaces
```

### DTOs vs Entities — a critical distinction

```typescript
// ENTITY — represents your DATABASE table
// Used internally by TypeORM/Prisma
// Contains ALL database columns including sensitive ones
@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @Column({ unique: true })
  email: string;

  @Column()
  password: string;      // ← Sensitive! Don't expose to client

  @Column()
  salt: string;          // ← Sensitive! Don't expose to client

  @Column({ default: false })
  isAdmin: boolean;      // ← Should never be set by client input

  @CreateDateColumn()
  createdAt: Date;
}

// CREATE DTO — what the client sends to CREATE a user
// Only the fields the client is ALLOWED to provide
export class CreateUserDto {
  name: string;          // Client can set this
  email: string;         // Client can set this
  password: string;      // Client sends plain password, service hashes it
  // NO isAdmin — client cannot make themselves admin!
  // NO id, createdAt — these are auto-generated
}

// UPDATE DTO — what the client sends to UPDATE a user
export class UpdateUserDto {
  name?: string;         // All optional — client only sends changed fields
  email?: string;
  // NO password — separate endpoint for password changes
}

// RESPONSE DTO — what you send BACK to the client
export class UserResponseDto {
  id: number;
  name: string;
  email: string;
  createdAt: Date;
  // NO password, NO salt, NO sensitive fields
}

// KEY RULE: You should almost always have SEPARATE DTOs for:
// - Creating (CreateUserDto)
// - Updating (UpdateUserDto)
// - Querying/filtering (QueryUserDto)
// - Response (UserResponseDto)
```

---

## 2. class-validator — Every Important Decorator

### Installation and setup

```bash
npm install class-validator class-transformer
```

### Complete reference — every decorator you need

```typescript
import {
  // String validators
  IsString, IsNotEmpty, IsEmail, IsUrl, IsUUID,
  MinLength, MaxLength, Matches, IsAlpha, IsAlphanumeric,
  Contains, NotContains, IsHexColor, IsMobilePhone,
  IsIP, IsJSON, IsBase64, IsCreditCard, IsIBAN,

  // Number validators
  IsNumber, IsInt, IsPositive, IsNegative,
  Min, Max, IsDivisibleBy, IsNumberString,

  // Boolean validators
  IsBoolean, IsBooleanString,

  // Date validators
  IsDate, IsDateString, MinDate, MaxDate,

  // Array validators
  IsArray, ArrayNotEmpty, ArrayMinSize, ArrayMaxSize,
  ArrayContains, ArrayNotContains, ArrayUnique,

  // Object validators
  IsObject, IsNotEmptyObject, ValidateNested,

  // Enum validators
  IsEnum,

  // Common validators
  IsOptional, IsNotIn, IsIn,
  Equals, NotEquals, IsDefined, IsEmpty,

  // Type checking
  IsInstance,
} from 'class-validator';

export class FullValidationExampleDto {

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // STRING VALIDATORS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  @IsString()
  // Must be a string type. Fails for numbers, booleans, arrays.
  // Always pair with @IsNotEmpty() if the field is required
  basicString: string;

  @IsNotEmpty()
  @IsString()
  // @IsNotEmpty(): rejects '', ' ', null, undefined
  // String must have at least 1 non-whitespace character
  name: string;

  @IsEmail()
  // Validates email format: must have @, domain, TLD
  // Accepts: "user@example.com", "user+tag@domain.co.uk"
  // Rejects: "notanemail", "@nodomain", "no@tld"
  email: string;

  @IsUrl({ protocols: ['https'], require_tld: true })
  // Validates URL format. Options let you restrict protocols.
  // require_tld: true = must have .com/.org/etc (no localhost)
  @IsOptional()
  website?: string;

  @IsUUID('4')
  // Validates UUID format. '4' = UUID v4 specifically.
  // Also accepts '3', '5', 'all'
  // Format: 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'
  @IsOptional()
  externalId?: string;

  @MinLength(8)
  @MaxLength(100)
  // MinLength: string must be at least N characters long
  // MaxLength: string must be at most N characters long
  @IsString()
  password: string;

  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/, {
    message: 'Password must contain uppercase, lowercase, number, and special character',
  })
  // Regex validation — most powerful string validator
  // This regex enforces strong password requirements
  @IsString()
  strongPassword: string;

  @IsAlpha()
  // Only alphabetical characters (a-z, A-Z). No numbers, spaces, or symbols.
  // Good for: first names, last names (simple use cases)
  @IsOptional()
  firstName?: string;

  @IsAlphanumeric()
  // Only letters and numbers. No spaces or symbols.
  // Good for: usernames, product codes
  @IsOptional()
  username?: string;

  @IsMobilePhone('en-IN')
  // Validates phone number format for a specific locale
  // 'en-IN' = Indian phone numbers
  // Use 'any' to accept all international formats
  @IsOptional()
  phone?: string;

  @IsHexColor()
  // Validates hex color: #fff, #ffffff, #AABBCC
  @IsOptional()
  brandColor?: string;

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // NUMBER VALIDATORS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  @IsInt()
  // Must be an INTEGER (no decimals). 5 ✅, 5.5 ❌
  // Note: JSON numbers are all floats internally — use @Transform to ensure int
  @Min(1)
  @Max(150)
  age: number;

  @IsNumber({ maxDecimalPlaces: 2 })
  // Allows decimal numbers. Options control decimal places.
  // Good for: prices, percentages, coordinates
  @IsPositive()  // Must be > 0. @IsNegative() for < 0
  @Min(0.01)
  price: number;

  @IsDivisibleBy(5)
  // Must be divisible by N with no remainder
  // Good for: batch sizes, quantities in multiples
  @IsInt()
  @IsOptional()
  batchSize?: number;

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // BOOLEAN VALIDATORS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  @IsBoolean()
  // Must be exactly true or false (boolean type)
  // Fails for 'true', 'false', 0, 1 (wrong types)
  // Use @IsBooleanString() for string 'true'/'false' (query params)
  @IsOptional()
  isActive?: boolean;

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // DATE VALIDATORS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  @IsDateString()
  // Validates ISO 8601 date string: '2024-01-01', '2024-01-01T10:00:00Z'
  // This is what you receive in JSON (JSON has no Date type)
  // Use @Type(() => Date) + @IsDate() if you want a Date object
  @IsOptional()
  birthDate?: string;

  @MinDate(new Date('2020-01-01'))
  @MaxDate(new Date())
  // Date must be after MinDate and before MaxDate
  // Use with @Type(() => Date) to convert string to Date first
  @IsDate()
  @IsOptional()
  eventDate?: Date;

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // ARRAY VALIDATORS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  @IsArray()
  @ArrayNotEmpty()              // Must have at least 1 element
  @ArrayMinSize(1)              // Same as ArrayNotEmpty but explicit
  @ArrayMaxSize(10)             // At most 10 elements
  @IsString({ each: true })     // each: true validates EACH element
  @IsNotEmpty({ each: true })   // Each string must not be empty
  tags: string[];

  @IsArray()
  @IsEnum(Role, { each: true }) // Each element must be a valid Role enum value
  @ArrayUnique()                // No duplicate values in array
  @IsOptional()
  roles?: Role[];

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // ENUM VALIDATORS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  @IsEnum(UserStatus, {
    message: `Status must be one of: ${Object.values(UserStatus).join(', ')}`,
  })
  // Only allows values defined in the UserStatus enum
  // Dynamic message shows allowed values — great for debugging
  status: UserStatus;

  @IsIn(['metric', 'imperial'])
  // Shortcut: value must be one of these specific strings
  // Good when you don't want a full enum
  @IsOptional()
  measurementSystem?: string;

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // CONDITIONAL VALIDATORS
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  @IsOptional()
  // If this field is not present → skip ALL validation on it
  // If present → run the other validators (@IsString, etc.)
  // Critical for UPDATE DTOs where all fields are optional
  optionalField?: string;

  @IsNotIn(['admin', 'root', 'system'])
  // Value must NOT be in this list
  // Good for: reserved usernames
  @IsString()
  notReservedUsername: string;
}
```

### Error messages — customizing validation feedback

```typescript
// Default error messages are developer-friendly but sometimes too technical
// You can customize every message

export class CreateUserDto {
  @IsNotEmpty({ message: 'Name is required' })
  @IsString({ message: 'Name must be text' })
  @MinLength(2, { message: 'Name must be at least 2 characters long' })
  @MaxLength(50, { message: 'Name cannot exceed 50 characters' })
  name: string;

  @IsEmail({}, { message: 'Please enter a valid email address' })
  email: string;

  @IsString({ message: 'Password must be a string' })
  @MinLength(8, { message: 'Password must be at least 8 characters' })
  @Matches(
    /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
    {
      // Message can be a function — receives ValidationArguments
      message: (args) =>
        `Password '${args.value}' is too weak. It must contain uppercase, lowercase, and a number.`,
    },
  )
  password: string;
}

// What the client receives on validation failure (400 Bad Request):
// {
//   "statusCode": 400,
//   "message": [
//     "Name is required",
//     "Please enter a valid email address",
//     "Password must be at least 8 characters"
//   ],
//   "error": "Bad Request"
// }
```

---

## 3. ValidationPipe — Deep Configuration

### Full configuration with every option explained

```typescript
// src/main.ts

import { ValidationPipe, BadRequestException } from '@nestjs/common';

app.useGlobalPipes(
  new ValidationPipe({
    // ── Core Options ────────────────────────────────────────────────────

    whitelist: true,
    // STRIPS properties not in the DTO class
    // Request: { name, email, isAdmin: true }
    // DTO has: name, email (no isAdmin)
    // After whitelist: { name, email } ← isAdmin silently removed
    // This prevents mass assignment attacks

    forbidNonWhitelisted: true,
    // Instead of silently stripping → THROWS 400 Bad Request
    // Client gets: "property isAdmin should not exist"
    // Recommended for development to catch API misuse early
    // Some teams turn this OFF in production for backward compatibility

    transform: true,
    // Converts plain JavaScript objects → DTO class instances
    // This is REQUIRED for class-validator to work!
    // Also enables automatic type coercion:
    // Query param '5' → number 5 (based on TS type annotation)
    // Query param 'true' → boolean true
    // Without this: @IsInt() on a query param always fails (strings don't validate as int)

    transformOptions: {
      enableImplicitConversion: true,
      // Even more aggressive: converts types based on TypeScript metadata
      // '5' → 5 for any field typed as number (even without @Transform)
      // 'true' → true for any field typed as boolean
      // Useful for query parameters which are always strings in HTTP
    },

    // ── Error Handling Options ──────────────────────────────────────────

    disableErrorMessages: false,
    // true: hides field names in error messages (security)
    // "name should not be empty" → "Validation failed"
    // Consider: true in production, false in development

    stopAtFirstError: false,
    // false (default): validates ALL fields, returns ALL errors at once
    // true: stops at first error found — returns only one error at a time
    // false is better UX: user can fix all errors at once

    skipMissingProperties: false,
    // false (default): validates ALL fields including undefined ones
    // true: skips validation on missing/undefined properties
    // Generally keep false — let @IsOptional() handle optional fields explicitly

    // ── Advanced Options ────────────────────────────────────────────────

    groups: [],
    // Validation groups — run only validators belonging to these groups
    // Advanced: use when same DTO has different rules for create vs update
    // Example: @IsNotEmpty({ groups: ['create'] })

    // ── Custom Exception Factory ────────────────────────────────────────
    exceptionFactory: (errors) => {
      // errors: ValidationError[] — array of all validation failures
      // Default factory throws BadRequestException with error messages

      // Custom factory: build a detailed error response
      const formattedErrors = errors.reduce((acc, error) => {
        // error.property: field name ('email')
        // error.constraints: { isEmail: 'email must be an email' }
        acc[error.property] = Object.values(error.constraints || {});
        return acc;
      }, {} as Record<string, string[]>);

      // Returns field-level errors instead of flat array
      // { "email": ["email must be an email"], "name": ["name is required"] }
      return new BadRequestException({
        statusCode: 400,
        message: 'Validation failed',
        errors: formattedErrors,
      });
    },
  }),
);
```

---

## 4. Nested Object Validation

### The problem with nested objects

```typescript
// ❌ Without @ValidateNested — nested validation is SKIPPED
export class CreateOrderDto {
  @IsString()
  product: string;

  address: AddressDto;  // ← class-validator IGNORES this!
  // Even if AddressDto has @IsNotEmpty() decorators
  // They are NOT checked unless you use @ValidateNested()
}

// ✅ With @ValidateNested — nested objects are fully validated
import { ValidateNested, Type } from 'class-validator';
import { Type } from 'class-transformer';  // @Type is from class-transformer

export class AddressDto {
  @IsString()
  @IsNotEmpty()
  street: string;

  @IsString()
  @IsNotEmpty()
  city: string;

  @IsString()
  @Length(2, 2)  // Exactly 2 characters (state codes)
  state: string;

  @Matches(/^\d{6}$/, { message: 'PIN code must be 6 digits' })
  pinCode: string;

  @IsString()
  @IsNotEmpty()
  country: string;
}

export class CreateOrderDto {
  @IsString()
  @IsNotEmpty()
  product: string;

  @IsInt()
  @Min(1)
  quantity: number;

  @ValidateNested()
  // Tells class-validator to go INTO this object and validate it
  // MUST be paired with @Type() from class-transformer

  @Type(() => AddressDto)
  // CRITICAL: This converts the plain JSON object → AddressDto instance
  // Without @Type(), the nested object is a plain object, not AddressDto
  // class-validator can only validate class instances with decorators
  address: AddressDto;
}
```

### Nested arrays of objects

```typescript
export class CreateInvoiceDto {
  @IsString()
  customerId: string;

  // Array of nested objects
  @IsArray()
  @ValidateNested({ each: true })
  // each: true → validate EACH element of the array
  @Type(() => InvoiceItemDto)
  // @Type() with each → converts each array element to InvoiceItemDto instance
  items: InvoiceItemDto[];
}

export class InvoiceItemDto {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsNumber({ maxDecimalPlaces: 2 })
  @IsPositive()
  price: number;

  @IsInt()
  @Min(1)
  quantity: number;
}

// Request body:
// {
//   "customerId": "cust-123",
//   "items": [
//     { "name": "Widget", "price": 9.99, "quantity": 2 },
//     { "name": "Gadget", "price": 24.99, "quantity": 1 }
//   ]
// }
// Each item is fully validated against InvoiceItemDto rules
```

### Multi-level nesting

```typescript
// Three levels deep — works the same way
export class CreateUserProfileDto {
  @ValidateNested()
  @Type(() => PersonalInfoDto)
  personalInfo: PersonalInfoDto;
}

export class PersonalInfoDto {
  @IsString()
  firstName: string;

  @ValidateNested()
  @Type(() => AddressDto)
  // Second level nesting
  address: AddressDto;
}

export class AddressDto {
  @IsString()
  city: string;

  @ValidateNested()
  @Type(() => CoordinatesDto)
  // Third level
  coordinates?: CoordinatesDto;
}

export class CoordinatesDto {
  @IsNumber()
  @Min(-90) @Max(90)
  latitude: number;

  @IsNumber()
  @Min(-180) @Max(180)
  longitude: number;
}
```

---

## 5. class-transformer — Type Conversion

### What is class-transformer?
`class-transformer` converts **plain JavaScript objects** (like JSON.parse output) into **typed class instances** and vice versa. It also lets you transform field values during conversion.

### Why it's needed alongside class-validator

```
Client sends JSON:
{ "age": "25", "isAdmin": "true", "createdAt": "2024-01-01" }

JSON.parse gives you plain object:
{ age: "25", isAdmin: "true", createdAt: "2024-01-01" }
          ↑ string     ↑ string         ↑ string

class-transformer converts to DTO instance:
{ age: 25, isAdmin: true, createdAt: Date(2024-01-01) }
       ↑ number   ↑ boolean           ↑ Date object

class-validator can now validate correctly:
@IsInt() → 25 ✅  (would fail on "25" string)
@IsBoolean() → true ✅
@IsDate() → Date ✅
```

### @Transform — the most powerful decorator

```typescript
import { Transform, Type, Exclude, Expose, plainToInstance, instanceToPlain } from 'class-transformer';

export class CreateUserDto {

  // ── Trim whitespace ────────────────────────────────────────────────
  @Transform(({ value }) => value?.trim())
  // Removes leading/trailing whitespace before validation
  // "  alice@email.com  " → "alice@email.com"
  @IsEmail()
  email: string;

  // ── Convert to lowercase ───────────────────────────────────────────
  @Transform(({ value }) => value?.trim().toLowerCase())
  // " ALICE " → "alice"
  // Normalize before saving to prevent duplicate users
  @IsString()
  username: string;

  // ── Convert string to number ───────────────────────────────────────
  @Transform(({ value }) => parseInt(value, 10))
  // '25' → 25
  // Useful for form data where everything comes as strings
  @IsInt()
  @Min(18)
  age: number;

  // ── Convert string to boolean ──────────────────────────────────────
  @Transform(({ value }) => {
    if (value === 'true' || value === true || value === 1) return true;
    if (value === 'false' || value === false || value === 0) return false;
    return value;  // Return as-is if can't convert — validator will catch it
  })
  @IsBoolean()
  isActive: boolean;

  // ── Convert string to Date ─────────────────────────────────────────
  @Type(() => Date)
  // @Type() with Date class: '2024-01-01' → new Date('2024-01-01')
  // Simpler than @Transform for type conversion
  @IsDate()
  @IsOptional()
  birthDate?: Date;

  // ── Parse JSON string ──────────────────────────────────────────────
  @Transform(({ value }) => {
    try {
      return typeof value === 'string' ? JSON.parse(value) : value;
    } catch {
      return value;  // Return as-is, let validator catch the error
    }
  })
  @IsObject()
  @IsOptional()
  metadata?: Record<string, any>;

  // ── Sanitize dangerous input ───────────────────────────────────────
  @Transform(({ value }) => {
    // Remove HTML tags to prevent XSS
    return typeof value === 'string'
      ? value.replace(/<[^>]*>/g, '')
      : value;
  })
  @IsString()
  @IsOptional()
  bio?: string;

  // ── Convert to array if single value ──────────────────────────────
  @Transform(({ value }) => (Array.isArray(value) ? value : [value]))
  // Handles both: tags=one AND tags=one&tags=two (HTTP query string behavior)
  // Without this: single value comes as string, multiple come as array
  @IsArray()
  @IsString({ each: true })
  @IsOptional()
  tags?: string[];

  // ── Normalize phone number ─────────────────────────────────────────
  @Transform(({ value }) => value?.replace(/[\s\-\(\)]/g, ''))
  // "(+91) 98765-43210" → "+919876543210"
  // Remove formatting, keep only digits and +
  @IsMobilePhone()
  @IsOptional()
  phone?: string;
}
```

### @Type — nested class conversion

```typescript
// @Type tells class-transformer: "when you encounter this field,
// convert it into an INSTANCE of this class"
// Required for @ValidateNested to work

export class OrderDto {
  @Type(() => Date)
  // '2024-01-01T10:00:00Z' → new Date('2024-01-01T10:00:00Z')
  @IsDate()
  scheduledAt: Date;

  @Type(() => AddressDto)
  // { street: '123 Main', city: 'Mumbai' } → new AddressDto instance
  @ValidateNested()
  deliveryAddress: AddressDto;

  @Type(() => OrderItemDto)
  // Each element in the array → new OrderItemDto instance
  @ValidateNested({ each: true })
  @IsArray()
  items: OrderItemDto[];
}
```

### plainToInstance and instanceToPlain

```typescript
// Two utility functions from class-transformer used frequently

// ── plainToInstance ──────────────────────────────────────────────────
// Converts a plain object (like from JSON) → a class instance
// This is what ValidationPipe does internally

import { plainToInstance } from 'class-transformer';

const plainObject = { name: 'Alice', email: 'alice@example.com', password: 'secret' };
const dto = plainToInstance(CreateUserDto, plainObject);
// dto is now a CreateUserDto instance with all @Transform decorators applied

// ── instanceToPlain ──────────────────────────────────────────────────
// Converts a class instance → plain object
// Respects @Exclude() and @Expose() decorators (covered in section 6)

import { instanceToPlain } from 'class-transformer';

const user = await this.usersService.findOne(1);
// user is a User entity with password field

const safeUser = instanceToPlain(user);
// If User entity has @Exclude() on password → safeUser won't have password field
```

---

## 6. Response Serialization — Hiding Sensitive Fields

### The problem

```typescript
// Your service returns the full User entity from the database
async findOne(id: number): Promise<User> {
  return this.userRepository.findOne({ where: { id } });
}

// This User entity has:
// { id, name, email, password, salt, resetToken, isAdmin, ... }

// If your controller returns this directly:
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  return this.usersService.findOne(id);
  // NestJS serializes this to JSON and sends it to client
  // Client receives password, salt, resetToken — SECURITY DISASTER
}
```

### Solution 1: @Exclude() on Entity (simplest)

```typescript
// src/users/entities/user.entity.ts

import { Exclude } from 'class-transformer';
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @Column({ unique: true })
  email: string;

  @Exclude()
  // This field will be EXCLUDED from JSON serialization
  // It still exists in the class — just hidden from output
  @Column()
  password: string;

  @Exclude()
  @Column({ nullable: true })
  resetPasswordToken: string;

  @Exclude()
  @Column({ nullable: true })
  twoFactorSecret: string;

  @Column({ default: false })
  isAdmin: boolean;

  @Column({ type: 'timestamp', default: () => 'CURRENT_TIMESTAMP' })
  createdAt: Date;
}
```

For `@Exclude()` to work, you MUST enable the ClassSerializerInterceptor:

```typescript
// src/main.ts OR in AppModule providers

import { ClassSerializerInterceptor } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

// In main.ts:
app.useGlobalInterceptors(new ClassSerializerInterceptor(app.get(Reflector)));

// OR in AppModule (recommended for DI support):
@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: ClassSerializerInterceptor,
      // ClassSerializerInterceptor calls instanceToPlain() on every response
      // This triggers @Exclude() and @Expose() decorators
    },
  ],
})
export class AppModule {}

// Now when controller returns a User entity:
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number): Promise<User> {
  return this.usersService.findOne(id);
  // ClassSerializerInterceptor intercepts this
  // Calls instanceToPlain(user) → excludes password, resetToken, twoFactorSecret
  // Client receives: { id, name, email, isAdmin, createdAt }
}
```

### Solution 2: @Expose() with excludeExtraneousValues (more controlled)

```typescript
// When you want to be explicit about WHICH fields are exposed
// rather than excluding individual fields

import { Expose, Exclude } from 'class-transformer';

// Create a separate response DTO (NOT the entity)
@Exclude()  // Exclude EVERYTHING by default
export class UserResponseDto {
  @Expose()  // Explicitly expose only these fields
  id: number;

  @Expose()
  name: string;

  @Expose()
  email: string;

  // No @Expose() on password → it's excluded (from @Exclude() on class)
  // This is "whitelist" approach — safer than excluding individually

  @Expose()
  createdAt: Date;

  constructor(partial: Partial<UserResponseDto>) {
    Object.assign(this, partial);
    // Convenience: let you do new UserResponseDto(user)
  }
}

// In the service, return the response DTO instead of the entity:
async findOne(id: number): Promise<UserResponseDto> {
  const user = await this.userRepository.findOne({ where: { id } });
  if (!user) throw new NotFoundException();

  return new UserResponseDto(user);
  // OR use plainToInstance with excludeExtraneousValues:
  // return plainToInstance(UserResponseDto, user, { excludeExtraneousValues: true });
}
```

### Solution 3: @SerializeOptions — per-route control

```typescript
import { SerializeOptions } from '@nestjs/common';

@Controller('users')
export class UsersController {

  // Show different fields based on route
  @Get(':id')
  @SerializeOptions({
    groups: ['public'],
    // Only expose fields decorated with @Expose({ groups: ['public'] })
  })
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.findOne(id);
  }

  @Get(':id/admin')
  @UseGuards(AdminGuard)
  @SerializeOptions({
    groups: ['admin'],
    // Admin route exposes more fields
  })
  findOneAdmin(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.findOne(id);
  }
}

// Entity with group-based exposure:
@Entity('users')
export class User {
  @Expose({ groups: ['public', 'admin'] })
  // Visible to BOTH public and admin
  @PrimaryGeneratedColumn()
  id: number;

  @Expose({ groups: ['public', 'admin'] })
  @Column()
  name: string;

  @Expose({ groups: ['admin'] })
  // ONLY visible to admin routes
  @Column()
  email: string;

  @Exclude()  // Always hidden
  @Column()
  password: string;

  @Expose({ groups: ['admin'] })
  @Column({ default: false })
  isAdmin: boolean;
}
```

### Solution 4: Manual mapping with class-transformer (most explicit)

```typescript
// The safest approach — you control every field explicitly

// In UsersService:
async findOne(id: number): Promise<UserResponseDto> {
  const user = await this.userRepository.findOne({ where: { id } });
  if (!user) throw new NotFoundException(`User #${id} not found`);

  // Manually map entity → response DTO
  // 100% explicit — no accidental field exposure
  return plainToInstance(UserResponseDto, {
    id: user.id,
    name: user.name,
    email: user.email,
    createdAt: user.createdAt,
    // Intentionally omitting: password, salt, resetToken, etc.
  }, {
    excludeExtraneousValues: true,
    // Strips any properties NOT decorated with @Expose() in UserResponseDto
  });
}
```

---

## 7. Whitelisting & Security Strategies

### Mass Assignment Attack — what it is and how to prevent it

```
Mass Assignment Attack:
  Client sends: { name: "Alice", email: "alice@test.com", isAdmin: true }
  Server blindly assigns all to entity: Object.assign(user, body)
  Result: Alice is now an admin!

This is a real vulnerability. NestJS's whitelist option prevents it.
```

```typescript
// Without protection — DANGEROUS
@Post()
create(@Body() body: any) {
  const user = new User();
  Object.assign(user, body);  // ← isAdmin gets set!
  return this.userRepository.save(user);
}

// With ValidationPipe whitelist — SAFE
// main.ts:
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,      // Strip unknown props
  forbidNonWhitelisted: true,  // Throw if unknown props sent
}));

// DTO only has safe fields:
export class CreateUserDto {
  @IsString() name: string;
  @IsEmail() email: string;
  @IsString() password: string;
  // NO isAdmin field → any isAdmin in request is stripped
}

// Controller — body is guaranteed to only contain DTO fields:
@Post()
create(@Body() createUserDto: CreateUserDto) {
  // createUserDto can ONLY have: name, email, password
  // isAdmin was stripped by whitelist before reaching here
  return this.usersService.create(createUserDto);
}
```

### Validation groups — different rules for different operations

```typescript
// Advanced pattern: same DTO class, different validation rules
// based on context (create vs update vs admin)

export class UserDto {
  @IsNotEmpty({ groups: ['create'] })
  // @IsNotEmpty only checked during 'create' group
  // During 'update', this field can be undefined
  @IsString()
  @IsOptional()  // Always optional for update
  name?: string;

  @IsEmail({ groups: ['create', 'update'] })
  // Email validated in both groups
  @IsNotEmpty({ groups: ['create'] })
  // But only required in create
  @IsOptional()
  email?: string;
}

// In controller, specify the group:
@Post()
create(
  @Body(new ValidationPipe({ groups: ['create'] })) dto: UserDto
) {}

@Patch(':id')
update(
  @Body(new ValidationPipe({ groups: ['update'] })) dto: UserDto
) {}
```

---

## 8. Mapped Types — DRY DTOs

### The DRY problem

```typescript
// Without Mapped Types — you repeat yourself
export class CreateUserDto {
  @IsString() @IsNotEmpty() name: string;
  @IsEmail() email: string;
  @IsString() @MinLength(8) password: string;
  @IsEnum(UserRole) role: UserRole;
}

export class UpdateUserDto {
  @IsString() @IsNotEmpty() @IsOptional() name?: string;
  @IsEmail() @IsOptional() email?: string;
  @IsString() @MinLength(8) @IsOptional() password?: string;
  @IsEnum(UserRole) @IsOptional() role?: UserRole;
  // Exact same validators but everything is optional — duplicated!
}
```

### PartialType — makes all fields optional

```typescript
import { PartialType } from '@nestjs/mapped-types';
// npm install @nestjs/mapped-types (usually included with NestJS)

export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;
}

// PartialType creates a new class where ALL fields from CreateUserDto
// become optional (as if you added @IsOptional() to each one)
// It INHERITS all the other validators (@IsString, @IsEmail, etc.)
export class UpdateUserDto extends PartialType(CreateUserDto) {}

// Result: UpdateUserDto has:
// name?: string (with @IsString, @IsNotEmpty if provided)
// email?: string (with @IsEmail if provided)
// password?: string (with @IsString, @MinLength(8) if provided)
// All validators still run IF the field is provided
```

### OmitType — exclude specific fields

```typescript
import { OmitType } from '@nestjs/mapped-types';

export class CreateUserDto {
  @IsString() name: string;
  @IsEmail() email: string;
  @IsString() @MinLength(8) password: string;
  @IsEnum(UserRole) role: UserRole;
}

// OmitType creates a new class that is CreateUserDto
// but WITHOUT the specified fields
export class CreateUserWithoutRoleDto extends OmitType(
  CreateUserDto,
  ['role'] as const,  // 'as const' for TypeScript to infer literal type
) {}
// Result: { name, email, password } — no role field

// Use case: Regular users can't set their own role
// Admin users use CreateUserDto (with role)
// Regular registration uses CreateUserWithoutRoleDto (without role)
```

### PickType — keep only specific fields

```typescript
import { PickType } from '@nestjs/mapped-types';

export class CreateUserDto {
  @IsString() name: string;
  @IsEmail() email: string;
  @IsString() password: string;
  @IsEnum(UserRole) role: UserRole;
  @IsUrl() profilePicture: string;
}

// PickType: only KEEP the specified fields
export class LoginDto extends PickType(
  CreateUserDto,
  ['email', 'password'] as const,
) {}
// Result: { email: string; password: string }
// Login only needs email and password
```

### IntersectionType — combine two DTOs

```typescript
import { IntersectionType } from '@nestjs/mapped-types';

export class BaseUserDto {
  @IsString() name: string;
  @IsEmail() email: string;
}

export class UserPreferencesDto {
  @IsEnum(Theme) theme: Theme;
  @IsString() language: string;
}

// IntersectionType combines both DTOs into one
export class CreateUserWithPreferencesDto extends IntersectionType(
  BaseUserDto,
  UserPreferencesDto,
) {}
// Result: { name, email, theme, language }
// All validators from both DTOs apply
```

### Composing Mapped Types

```typescript
// You can CHAIN mapped type utilities
export class CreateUserDto {
  @IsString() name: string;
  @IsEmail() email: string;
  @IsString() password: string;
  @IsEnum(UserRole) role: UserRole;
  @IsString() createdBy: string;
}

// Update: all optional, but can't change role or createdBy
export class UpdateUserDto extends PartialType(
  OmitType(CreateUserDto, ['role', 'createdBy'] as const)
) {}
// Result: { name?: string; email?: string; password?: string }
// role and createdBy are completely removed
// Remaining fields are all made optional by PartialType
```

---

## 9. Custom Validation Decorators

### Simple custom decorator with registerDecorator

```typescript
// src/common/validators/is-strong-password.validator.ts

import {
  registerDecorator,
  ValidationOptions,
  ValidationArguments,
} from 'class-validator';

// Factory function that returns a decorator
export function IsStrongPassword(validationOptions?: ValidationOptions) {
  return function (object: Object, propertyName: string) {
    // registerDecorator does the heavy lifting
    registerDecorator({
      name: 'isStrongPassword',  // Name used in error messages
      target: object.constructor,
      propertyName: propertyName,
      options: validationOptions,
      validator: {
        // validate() returns true if valid, false if invalid
        validate(value: any, args: ValidationArguments): boolean {
          if (typeof value !== 'string') return false;

          const hasMinLength = value.length >= 8;
          const hasUppercase = /[A-Z]/.test(value);
          const hasLowercase = /[a-z]/.test(value);
          const hasNumber = /\d/.test(value);
          const hasSpecialChar = /[@$!%*?&]/.test(value);

          return hasMinLength && hasUppercase && hasLowercase
            && hasNumber && hasSpecialChar;
        },

        // defaultMessage() called if no custom message provided
        defaultMessage(args: ValidationArguments): string {
          return `${args.property} must be at least 8 characters with uppercase, lowercase, number, and special character (@$!%*?&)`;
        },
      },
    });
  };
}

// Usage:
export class CreateUserDto {
  @IsStrongPassword({ message: 'Please choose a stronger password' })
  password: string;
}
```

### Custom decorator with Injectable (uses DI)

```typescript
// src/common/validators/is-unique-email.validator.ts
// Problem: Check if email already exists in database — needs a service

import {
  registerDecorator,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface,
  ValidationArguments,
} from 'class-validator';
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from '../../users/entities/user.entity';

// Injectable validator — can use DI
@ValidatorConstraint({ name: 'isUniqueEmail', async: true })
// async: true because we need to query the database
@Injectable()
export class IsUniqueEmailConstraint implements ValidatorConstraintInterface {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  async validate(email: string): Promise<boolean> {
    const user = await this.userRepository.findOne({ where: { email } });
    return !user;  // true = valid (no existing user), false = invalid (email taken)
  }

  defaultMessage(args: ValidationArguments): string {
    return `Email '${args.value}' is already registered`;
  }
}

// The decorator factory
export function IsUniqueEmail(validationOptions?: ValidationOptions) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName: propertyName,
      options: validationOptions,
      constraints: [],
      validator: IsUniqueEmailConstraint,  // Reference the class, not instance
    });
  };
}

// Register the constraint in AppModule (so DI works):
@Module({
  providers: [IsUniqueEmailConstraint],  // Must be in providers for DI
})
export class AppModule {}

// Also enable useContainer in main.ts:
import { useContainer } from 'class-validator';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  useContainer(app.select(AppModule), { fallbackOnErrors: true });
  // Tells class-validator to use NestJS's DI container
  // Without this, @Injectable() validators can't get their dependencies
  await app.listen(3000);
}

// Usage in DTO:
export class CreateUserDto {
  @IsUniqueEmail({ message: 'This email is already in use' })
  @IsEmail()
  email: string;
}
```

---

## 10. Real-World DTO Patterns

### Query DTO — for filtering, pagination, sorting

```typescript
// src/users/dto/query-user.dto.ts

import { IsOptional, IsInt, Min, Max, IsEnum, IsString } from 'class-validator';
import { Type, Transform } from 'class-transformer';

export enum UserSortField {
  NAME = 'name',
  EMAIL = 'email',
  CREATED_AT = 'createdAt',
}

export enum SortOrder {
  ASC = 'ASC',
  DESC = 'DESC',
}

export class QueryUserDto {
  // Pagination
  @IsOptional()
  @Type(() => Number)    // Query params are strings — convert to number
  @IsInt()
  @Min(1)
  page?: number = 1;     // Default value = 1

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)              // Cap at 100 to prevent large queries
  limit?: number = 20;

  // Filtering
  @IsOptional()
  @IsString()
  @Transform(({ value }) => value?.trim())
  search?: string;       // Search by name or email

  @IsOptional()
  @IsEnum(UserStatus)
  status?: UserStatus;

  // Sorting
  @IsOptional()
  @IsEnum(UserSortField)
  sortBy?: UserSortField = UserSortField.CREATED_AT;

  @IsOptional()
  @IsEnum(SortOrder)
  sortOrder?: SortOrder = SortOrder.DESC;
}

// Usage in controller:
@Get()
findAll(@Query() query: QueryUserDto) {
  // GET /users?page=2&limit=10&search=alice&status=active&sortBy=name&sortOrder=ASC
  return this.usersService.findAll(query);
}

// Usage in service:
async findAll(query: QueryUserDto) {
  const { page, limit, search, status, sortBy, sortOrder } = query;

  const queryBuilder = this.userRepository.createQueryBuilder('user');

  if (search) {
    queryBuilder.where(
      'user.name ILIKE :search OR user.email ILIKE :search',
      { search: `%${search}%` },
    );
  }

  if (status) {
    queryBuilder.andWhere('user.status = :status', { status });
  }

  queryBuilder
    .orderBy(`user.${sortBy}`, sortOrder)
    .skip((page - 1) * limit)
    .take(limit);

  const [data, total] = await queryBuilder.getManyAndCount();

  return {
    data,
    meta: { total, page, limit, lastPage: Math.ceil(total / limit) },
  };
}
```

### Complete User DTOs — production example

```typescript
// src/users/dto/index.ts — barrel export of all DTOs

// ── Create ─────────────────────────────────────────────────────────────
export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  @MinLength(2)
  @MaxLength(50)
  @Transform(({ value }) => value?.trim())
  name: string;

  @IsEmail()
  @Transform(({ value }) => value?.toLowerCase().trim())
  email: string;

  @IsString()
  @MinLength(8)
  @MaxLength(128)
  password: string;

  @IsEnum(UserRole)
  @IsOptional()
  role?: UserRole = UserRole.USER;
}

// ── Update ─────────────────────────────────────────────────────────────
export class UpdateUserDto extends PartialType(
  OmitType(CreateUserDto, ['password'] as const)
  // Can update name, email, role — NOT password
  // Password change has its own endpoint with old/new password fields
) {}

// ── Change Password ─────────────────────────────────────────────────────
export class ChangePasswordDto {
  @IsString()
  @IsNotEmpty()
  currentPassword: string;

  @IsString()
  @MinLength(8)
  @MaxLength(128)
  newPassword: string;

  @IsString()
  confirmNewPassword: string;
}

// ── Response ─────────────────────────────────────────────────────────────
@Exclude()  // Exclude everything by default
export class UserResponseDto {
  @Expose() id: number;
  @Expose() name: string;
  @Expose() email: string;
  @Expose() role: UserRole;
  @Expose() isActive: boolean;
  @Expose() createdAt: Date;
  @Expose() updatedAt: Date;
  // password, salt, resetToken, etc. — NOT exposed

  constructor(partial: Partial<UserResponseDto>) {
    Object.assign(this, partial);
  }
}
```

---

## 11. Interview Questions & Answers

**Q1: What is a DTO and why is it a class instead of an interface in NestJS?**

> "A DTO (Data Transfer Object) defines the shape of data coming into or out of the API — it's a contract that enforces what fields are allowed, their types, and constraints. NestJS uses classes instead of interfaces because interfaces are a TypeScript construct that is erased at compile time and doesn't exist at runtime. `class-validator` needs to inspect the actual class and its decorator metadata at runtime to perform validation. Classes exist at runtime as real JavaScript constructor functions with prototype chains, allowing `ValidationPipe` to instantiate them and `class-validator` to read their metadata. Interfaces literally don't exist when your code runs."

---

**Q2: What is the difference between whitelist and forbidNonWhitelisted in ValidationPipe?**

> "Both options relate to properties in the request body that aren't declared in the DTO. `whitelist: true` silently strips those extra properties before the data reaches the controller — so if a client sends `isAdmin: true` and it's not in the DTO, it's simply removed. `forbidNonWhitelisted: true` goes further: instead of silently removing, it throws a 400 Bad Request with a message like 'property isAdmin should not exist'. In development, `forbidNonWhitelisted` is useful because it catches API misuse early. In production, some teams use only `whitelist` for softer behavior that doesn't break clients who send extra fields. Both together prevent mass assignment attacks where clients try to set fields like `isAdmin` or `role` that they shouldn't be able to control."

---

**Q3: What is ClassSerializerInterceptor and how does it work?**

> "It's a built-in NestJS interceptor that automatically calls `instanceToPlain()` from `class-transformer` on every response. This triggers any `@Exclude()` and `@Expose()` decorators on your response classes. You register it globally via `APP_INTERCEPTOR` in your AppModule. Without it, returning a User entity that has `@Exclude()` on the password field would still send the password to the client — because NestJS would just call `JSON.stringify()` which ignores those decorators. With the interceptor, every response goes through class-transformer serialization first, respecting all exclusion rules."

---

**Q4: How do you validate nested objects in NestJS?**

> "You use `@ValidateNested()` from `class-validator` combined with `@Type()` from `class-transformer`. `@ValidateNested()` tells the validator to recurse into the nested object and apply its validation decorators. But it needs `@Type(() => NestedClass)` because the incoming JSON is a plain object — class-validator can only validate class instances with decorators, not plain objects. `@Type()` converts the plain object into an instance of the specified class, making the decorators discoverable. For arrays of nested objects, you use `@ValidateNested({ each: true })` and `@Type(() => NestedClass)` — the `each: true` tells it to validate each element of the array."

---

**Q5: What is the difference between @Exclude() and @Expose() and when do you use each?**

> "They're two approaches to the same goal — controlling which fields appear in serialized output — but with opposite defaults. `@Exclude()` on a property hides that specific field, while everything else remains visible (blacklist approach). `@Exclude()` on the class itself hides everything, and you then use `@Expose()` on properties you want visible (whitelist approach). The whitelist approach with `@Exclude()` on the class and `@Expose()` on allowed fields is safer for sensitive data like User entities — you explicitly opt into exposing fields rather than trying to remember to exclude every sensitive field. You must enable `ClassSerializerInterceptor` for either approach to work."

---

**Q6: How do PartialType, OmitType, PickType, and IntersectionType work?**

> "These are utilities from `@nestjs/mapped-types` that create new DTO classes by transforming existing ones, keeping you DRY. `PartialType(CreateDto)` creates a new class where all fields from CreateDto are optional — perfect for update DTOs. `OmitType(CreateDto, ['field1', 'field2'])` creates a class without the specified fields — useful for hiding fields like `role` or `id` from certain operations. `PickType(CreateDto, ['email', 'password'])` creates a class with ONLY the specified fields — perfect for login DTOs. `IntersectionType(DtoA, DtoB)` merges two DTOs into one. Critically, all these utilities preserve the validation decorators from the source class — you don't have to rewrite `@IsEmail()` or `@MinLength()` in the derived class."

---

**Q7: How do you handle validation for query parameters that are always strings in HTTP?**

> "HTTP query parameters are always strings by nature — even `?page=5` comes as the string `'5'`, not the number `5`. There are two ways to handle this. First, use `@Type(() => Number)` from `class-transformer` in your query DTO — this explicitly converts the string to a number before validation. Second, enable `enableImplicitConversion: true` in `ValidationPipe`'s `transformOptions` — this automatically converts based on TypeScript type annotations, so a field typed as `number` gets its string value converted automatically. Both require `transform: true` in `ValidationPipe`. The explicit `@Type()` approach is clearer and preferred because it's obvious what conversion happens."

---

## Quick Reference — Day 4 Cheat Sheet

```
DTO Rule: Class (not interface) — class-validator needs runtime existence

ValidationPipe options:
  whitelist: true              → strip unknown fields silently
  forbidNonWhitelisted: true   → throw 400 for unknown fields
  transform: true              → convert types + plain objects to class instances
  enableImplicitConversion     → auto-convert based on TS types

class-validator quick reference:
  Strings:  @IsString @IsEmail @IsUrl @IsUUID @MinLength @MaxLength @Matches
  Numbers:  @IsInt @IsNumber @Min @Max @IsPositive @IsDivisibleBy
  Booleans: @IsBoolean
  Dates:    @IsDate @IsDateString @MinDate @MaxDate
  Arrays:   @IsArray @ArrayNotEmpty @ArrayMinSize @ArrayMaxSize @ArrayUnique
  Special:  @IsOptional @IsEnum @IsIn @IsNotIn @ValidateNested @IsDefined

Nested objects: @ValidateNested() + @Type(() => NestedClass) [BOTH required]

class-transformer:
  @Transform(({ value }) => ...)   → custom value transformation
  @Type(() => TargetClass)         → convert plain object to class instance
  @Exclude()                       → hide from serialization
  @Expose()                        → show in serialization (when class is @Exclude'd)

Mapped types (keep DTOs DRY):
  PartialType(Dto)            → all fields optional
  OmitType(Dto, ['f1','f2'])  → remove specific fields
  PickType(Dto, ['f1','f2'])  → keep only specific fields
  IntersectionType(A, B)      → merge two DTOs

ClassSerializerInterceptor:
  Must be enabled for @Exclude/@Expose to work on responses
  Register: { provide: APP_INTERCEPTOR, useClass: ClassSerializerInterceptor }

Entity vs DTO:
  Entity   → database table columns (ALL data, including sensitive)
  DTO      → API contract (only what client should send/receive)
  NEVER return entity directly — always control output via response DTO or @Exclude
```

---

*Day 4 complete. Tomorrow — Day 5: Configuration & Environment — ConfigModule, ConfigService, environment variables, Joi validation, typed config with registerAs(), and secrets management.*
