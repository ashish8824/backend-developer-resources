# Day 13 — Testing: Unit & End-to-End
### NestJS Interview Prep | 3-Year Backend Engineer Level

> **Goal for today:** Master testing in NestJS — unit tests for services and controllers, mocking providers with Jest, integration tests, e2e tests with Supertest, TestingModule setup, and code coverage. A 3-year engineer writes tests as naturally as they write code. Interviewers WILL ask about testing strategies, what to test, and how to mock dependencies.

---

## 🧭 First-Timer Guidance — Read This First

**Why testing matters in interviews:**

When an interviewer asks "how do you ensure code quality?", the answer is NOT "I test it manually in Postman." The answer is a comprehensive testing strategy. Companies with engineering maturity require:
- Unit tests that run in milliseconds
- Integration tests for database operations
- E2E tests for critical user flows
- CI/CD pipeline that blocks deployments if tests fail

**The testing pyramid:**

```
           /\
          /  \
         / E2E \         Few, slow, expensive
        /________\       Test whole system
       /          \
      / Integration\     Some, medium speed
     /______________\    Test with real DB/services
    /                \
   /   Unit Tests     \  Many, fast, isolated
  /____________________\ Test one thing at a time

Rule: More unit tests, fewer e2e tests
```

**What to test at each level:**

```
Unit Tests:
  Service methods (business logic)
  Guards (canActivate logic)
  Interceptors (transform logic)
  Pipes (validation/transformation)
  Utilities and helpers
  
  Mock: everything external (DB, other services, external APIs)
  Don't mock: the thing you're testing

Integration Tests:
  Controller → Service → Repository (with real DB)
  Using TestingModule + in-memory SQLite or test PostgreSQL

E2E Tests:
  HTTP request → full stack → HTTP response
  POST /auth/login → JWT → GET /users (protected)
  Test complete user flows, not individual units
```

**Setup — NestJS already includes Jest:**
```bash
# Already configured when you run 'nest new'
# package.json has: "test": "jest", "test:e2e": "jest --config jest-e2e.json"

# Run tests:
npm test              # Unit tests (watch mode: npm run test:watch)
npm run test:cov      # With coverage report
npm run test:e2e      # End-to-end tests
```

---

## Table of Contents

1. [Jest Fundamentals — Before NestJS Testing](#1-jest-fundamentals--before-nestjs-testing)
2. [TestingModule — NestJS's Test Factory](#2-testingmodule--nestjss-test-factory)
3. [Testing Services — The Core Logic](#3-testing-services--the-core-logic)
4. [Mocking — Replacing Real Dependencies](#4-mocking--replacing-real-dependencies)
5. [Testing Controllers](#5-testing-controllers)
6. [Testing Guards, Pipes, and Interceptors](#6-testing-guards-pipes-and-interceptors)
7. [Integration Tests — With Real Database](#7-integration-tests--with-real-database)
8. [E2E Tests — Full Stack with Supertest](#8-e2e-tests--full-stack-with-supertest)
9. [Testing Auth — JWT and Guards](#9-testing-auth--jwt-and-guards)
10. [Code Coverage — What It Means](#10-code-coverage--what-it-means)
11. [Testing Best Practices](#11-testing-best-practices)
12. [Interview Questions & Answers](#12-interview-questions--answers)

---

## 1. Jest Fundamentals — Before NestJS Testing

### Core Jest concepts you must know

```typescript
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// TEST STRUCTURE
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

describe('UsersService', () => {
  // describe(): groups related tests — like a test suite
  // Name it after what you're testing: class, function, feature

  describe('findOne()', () => {
    // Nested describe: group by method or scenario
    // Makes test output readable:
    // UsersService
    //   findOne()
    //     ✓ should return user when found
    //     ✓ should throw NotFoundException when user not found

    it('should return user when found', () => {
      // it() or test(): individual test case
      // Name it: "should [expected behavior] when [condition]"
      // Short, specific, readable

      // Arrange → Act → Assert (AAA pattern)
      const expected = { id: 1, email: 'alice@test.com' };  // Arrange
      const result = userService.findOne(1);                  // Act
      expect(result).toEqual(expected);                       // Assert
    });

    test('should throw NotFoundException when user not found', async () => {
      // test() is identical to it() — both work
      // Use whichever reads more naturally
      await expect(userService.findOne(999))
        .rejects.toThrow(NotFoundException);
    });
  });
});

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// LIFECYCLE HOOKS
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

describe('Test Suite', () => {
  beforeAll(async () => {
    // Runs ONCE before all tests in this describe block
    // Use for: creating the TestingModule, connecting to DB
    // Expensive setup that can be shared across tests
  });

  afterAll(async () => {
    // Runs ONCE after all tests in this describe block
    // Use for: closing DB connections, cleanup
  });

  beforeEach(async () => {
    // Runs BEFORE EACH test
    // Use for: resetting mocks, creating fresh data per test
    // Ensures tests don't affect each other
    jest.clearAllMocks();
    // Clear mock call counts and return values between tests
  });

  afterEach(async () => {
    // Runs AFTER EACH test
    // Use for: cleanup specific to each test
    // Rolling back DB transactions after each test
  });
});

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// EXPECT MATCHERS — the assertion library
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

// Equality
expect(result).toBe(42);              // Strict equality (===) — use for primitives
expect(result).toEqual({ id: 1 });   // Deep equality — use for objects/arrays
expect(result).not.toBe(null);       // Negation with .not

// Type checks
expect(result).toBeDefined();        // Not undefined
expect(result).toBeNull();           // Strict null
expect(result).toBeTruthy();         // Any truthy value
expect(result).toBeFalsy();          // Any falsy value
expect(result).toBeInstanceOf(User); // instanceof check

// Numbers
expect(value).toBeGreaterThan(5);
expect(value).toBeLessThanOrEqual(10);
expect(value).toBeCloseTo(0.1 + 0.2); // Floating point comparison

// Strings
expect(str).toContain('hello');       // Substring
expect(str).toMatch(/^Hello/);        // Regex

// Arrays/Objects
expect(arr).toHaveLength(3);
expect(arr).toContain('item');
expect(obj).toHaveProperty('name', 'Alice');
expect(obj).toMatchObject({ id: 1 }); // Partial match — object contains these props

// Functions/Promises
expect(fn).toThrow('Error message');
expect(fn).toThrow(NotFoundException);
await expect(asyncFn()).resolves.toEqual({ id: 1 });
await expect(asyncFn()).rejects.toThrow('Error');
await expect(asyncFn()).rejects.toMatchObject({ status: 404 });

// Mock assertions (covered in section 4)
expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledTimes(2);
expect(mockFn).toHaveBeenCalledWith(arg1, arg2);
expect(mockFn).toHaveBeenLastCalledWith(arg1);
```

---

## 2. TestingModule — NestJS's Test Factory

### What is TestingModule?

`TestingModule` is NestJS's way to create an isolated version of your module for testing. It behaves like a real NestJS module but without starting an HTTP server.

```typescript
// The SAME way NestFactory.create(AppModule) creates your app,
// Test.createTestingModule() creates a test version with:
// - Full dependency injection
// - All providers instantiated
// - No HTTP server
// - No real DB (unless you want it)

import { Test, TestingModule } from '@nestjs/testing';

describe('UsersService', () => {
  let service: UsersService;
  let module: TestingModule;

  beforeEach(async () => {
    module = await Test.createTestingModule({
      // Same structure as @Module() decorator
      providers: [
        UsersService,
        // The service we're testing — REAL implementation

        // Dependencies of UsersService — MOCKED
        {
          provide: getRepositoryToken(User),
          // getRepositoryToken(User): the injection token TypeORM uses
          // Same as what @InjectRepository(User) resolves to
          useValue: mockUserRepository,
          // Provide a mock object instead of real TypeORM repository
        },

        {
          provide: ConfigService,
          useValue: {
            get: jest.fn().mockReturnValue('test-value'),
            getOrThrow: jest.fn().mockReturnValue('test-value'),
          },
        },
      ],
    }).compile();
    // .compile(): finalizes the module — resolves all dependencies

    service = module.get<UsersService>(UsersService);
    // module.get(): retrieve a provider from the test module
    // Returns the same singleton instance used during tests
  });

  afterEach(async () => {
    await module.close();
    // Clean up — close connections if any
  });
});
```

---

## 3. Testing Services — The Core Logic

### Complete service test file

```typescript
// src/users/users.service.spec.ts
// .spec.ts: Jest automatically finds and runs these files

import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { NotFoundException, ConflictException } from '@nestjs/common';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

// ── Mock factory — create a fresh mock repository for each test ────────────
const mockUserRepository = () => ({
  find: jest.fn(),
  findOne: jest.fn(),
  findAndCount: jest.fn(),
  create: jest.fn(),
  save: jest.fn(),
  update: jest.fn(),
  remove: jest.fn(),
  exists: jest.fn(),
  createQueryBuilder: jest.fn(),
});
// Using a factory function (not a plain object) ensures each test
// gets a FRESH mock with no state from previous tests

// ── Type alias for the mock ───────────────────────────────────────────────
type MockRepository<T = any> = Partial<Record<keyof Repository<T>, jest.Mock>>;

describe('UsersService', () => {
  let service: UsersService;
  let userRepository: MockRepository<User>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useFactory: mockUserRepository,
          // useFactory: creates a new instance per test
          // Better than useValue (shared state between tests)
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    userRepository = module.get<MockRepository<User>>(getRepositoryToken(User));
  });

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // SERVICE INSTANTIATION
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  it('should be defined', () => {
    expect(service).toBeDefined();
    // Basic sanity check — service was created successfully
    // If this fails: DI setup is wrong
  });

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // TESTING findAll()
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  describe('findAll()', () => {
    it('should return paginated users', async () => {
      // Arrange: set up what the mock should return
      const mockUsers: User[] = [
        { id: 1, email: 'alice@test.com', firstName: 'Alice', lastName: 'Smith' } as User,
        { id: 2, email: 'bob@test.com', firstName: 'Bob', lastName: 'Jones' } as User,
      ];
      const total = 10;

      userRepository.findAndCount.mockResolvedValue([mockUsers, total]);
      // mockResolvedValue: mock async function to resolve with this value
      // Like: jest.fn().mockImplementation(() => Promise.resolve([mockUsers, total]))

      // Act: call the service method
      const result = await service.findAll({ page: 1, limit: 2 });

      // Assert: verify the result
      expect(result.data).toEqual(mockUsers);
      expect(result.meta.total).toBe(total);
      expect(result.meta.page).toBe(1);
      expect(result.meta.limit).toBe(2);
      expect(result.meta.totalPages).toBe(5); // Math.ceil(10/2)

      // Verify the mock was called correctly
      expect(userRepository.findAndCount).toHaveBeenCalledWith({
        skip: 0,   // page 1 = skip 0
        take: 2,
        order: { createdAt: 'DESC' },
      });
    });

    it('should use correct offset for page 2', async () => {
      userRepository.findAndCount.mockResolvedValue([[], 50]);

      await service.findAll({ page: 2, limit: 10 });

      expect(userRepository.findAndCount).toHaveBeenCalledWith(
        expect.objectContaining({
          skip: 10,  // page 2 = skip 10
          take: 10,
        }),
      );
      // expect.objectContaining(): partial match
      // Checks that the argument CONTAINS these props (ignores others)
    });
  });

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // TESTING findOne()
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  describe('findOne()', () => {
    const mockUser: User = {
      id: 1,
      email: 'alice@test.com',
      firstName: 'Alice',
      lastName: 'Smith',
      role: 'user',
      isActive: true,
      createdAt: new Date('2024-01-01'),
    } as User;

    it('should return user when found', async () => {
      // Arrange
      userRepository.findOne.mockResolvedValue(mockUser);

      // Act
      const result = await service.findOne(1);

      // Assert
      expect(result).toEqual(mockUser);
      expect(userRepository.findOne).toHaveBeenCalledWith({
        where: { id: 1 },
      });
    });

    it('should throw NotFoundException when user not found', async () => {
      // Arrange: repository returns null (no user found)
      userRepository.findOne.mockResolvedValue(null);

      // Assert: service throws NotFoundException
      await expect(service.findOne(999)).rejects.toThrow(NotFoundException);
      await expect(service.findOne(999)).rejects.toThrow('User #999 not found');
      // Second assertion checks the error MESSAGE too

      // Alternative:
      await expect(service.findOne(999)).rejects.toMatchObject({
        message: 'User #999 not found',
        // status: 404  ← also works if you want to verify status code
      });
    });
  });

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // TESTING create()
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  describe('create()', () => {
    const createDto: CreateUserDto = {
      firstName: 'Alice',
      lastName: 'Smith',
      email: 'alice@test.com',
      password: 'SecurePass@123',
    };

    it('should create and return a new user', async () => {
      // Arrange
      const savedUser = { id: 1, ...createDto, createdAt: new Date() } as User;

      userRepository.findOne.mockResolvedValue(null);
      // No existing user with this email — can create

      userRepository.create.mockReturnValue(savedUser);
      // Note: .create() is SYNC (not async) — use mockReturnValue not mockResolvedValue

      userRepository.save.mockResolvedValue(savedUser);
      // .save() is ASYNC — use mockResolvedValue

      // Act
      const result = await service.create(createDto);

      // Assert
      expect(result).toEqual(savedUser);
      expect(userRepository.findOne).toHaveBeenCalledWith({
        where: { email: createDto.email },
      });
      expect(userRepository.create).toHaveBeenCalledWith(
        expect.objectContaining({
          email: createDto.email,
          firstName: createDto.firstName,
          // Don't check password here — it should be HASHED
          // The hash changes every time (different salt)
        }),
      );
      expect(userRepository.save).toHaveBeenCalledTimes(1);
    });

    it('should throw ConflictException when email already exists', async () => {
      // Arrange: email already taken
      userRepository.findOne.mockResolvedValue({
        id: 99,
        email: createDto.email,
      } as User);

      // Assert
      await expect(service.create(createDto)).rejects.toThrow(ConflictException);

      // Ensure user was NOT created
      expect(userRepository.create).not.toHaveBeenCalled();
      expect(userRepository.save).not.toHaveBeenCalled();
    });

    it('should hash the password before saving', async () => {
      // Arrange
      userRepository.findOne.mockResolvedValue(null);
      userRepository.create.mockImplementation((dto) => dto);
      // mockImplementation: custom function for the mock
      // Here: just return what was passed in

      userRepository.save.mockImplementation(async (user) => ({
        ...user,
        id: 1,
      }));

      // Act
      await service.create(createDto);

      // Assert: password was hashed (not stored as plain text)
      const createCall = userRepository.create.mock.calls[0][0];
      // .mock.calls: array of all calls to this mock
      // .mock.calls[0]: first call
      // .mock.calls[0][0]: first argument of first call

      expect(createCall.password).toBeDefined();
      expect(createCall.password).not.toBe(createDto.password);
      // Password stored is different from input → it was hashed
      expect(createCall.password).toMatch(/^\$2[aby]\$/);
      // bcrypt hashes always start with $2a$, $2b$, or $2y$
    });
  });

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // TESTING update()
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  describe('update()', () => {
    const existingUser: User = {
      id: 1,
      email: 'alice@test.com',
      firstName: 'Alice',
      lastName: 'Smith',
    } as User;

    it('should update and return the user', async () => {
      const updateDto: UpdateUserDto = { firstName: 'Alicia' };

      userRepository.findOne.mockResolvedValue(existingUser);
      userRepository.save.mockResolvedValue({
        ...existingUser,
        firstName: 'Alicia',
      });

      const result = await service.update(1, updateDto);

      expect(result.firstName).toBe('Alicia');
      expect(userRepository.save).toHaveBeenCalledWith(
        expect.objectContaining({
          id: 1,
          firstName: 'Alicia',
          email: 'alice@test.com', // Unchanged
        }),
      );
    });

    it('should throw NotFoundException when user does not exist', async () => {
      userRepository.findOne.mockResolvedValue(null);

      await expect(service.update(999, { firstName: 'Test' }))
        .rejects.toThrow(NotFoundException);
    });
  });

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // TESTING remove()
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  describe('remove()', () => {
    it('should remove the user successfully', async () => {
      const user = { id: 1, email: 'alice@test.com' } as User;

      userRepository.findOne.mockResolvedValue(user);
      userRepository.remove.mockResolvedValue(user);

      await service.remove(1);

      expect(userRepository.findOne).toHaveBeenCalledWith({ where: { id: 1 } });
      expect(userRepository.remove).toHaveBeenCalledWith(user);
    });

    it('should throw NotFoundException for non-existent user', async () => {
      userRepository.findOne.mockResolvedValue(null);

      await expect(service.remove(999)).rejects.toThrow(NotFoundException);
      expect(userRepository.remove).not.toHaveBeenCalled();
    });
  });
});
```

---

## 4. Mocking — Replacing Real Dependencies

### jest.fn() — the core mocking primitive

```typescript
// jest.fn(): creates a mock function that tracks calls
const mockFn = jest.fn();

// Setting return values:
mockFn.mockReturnValue('sync value');
// → every call returns 'sync value' (synchronous)

mockFn.mockResolvedValue({ id: 1 });
// → returns Promise.resolve({ id: 1 }) (async)

mockFn.mockRejectedValue(new Error('DB error'));
// → returns Promise.reject(new Error('DB error')) (async error)

mockFn.mockReturnValueOnce('first call')
      .mockReturnValueOnce('second call')
      .mockReturnValue('all subsequent calls');
// Different value per call — for testing sequences

mockFn.mockImplementation((arg1, arg2) => {
  // Custom function logic
  return arg1 + arg2;
});
// Full control over mock behavior

// Inspecting calls:
mockFn.mock.calls                   // [[arg1, arg2], [arg3, arg4], ...]
mockFn.mock.calls.length            // Number of times called
mockFn.mock.calls[0]                // Arguments of first call
mockFn.mock.calls[0][0]             // First argument of first call
mockFn.mock.results                 // [{ type: 'return', value: ... }, ...]
mockFn.mock.instances               // 'this' values (for class methods)

// Resetting mocks:
mockFn.mockClear();     // Clear calls/results but keep implementation
mockFn.mockReset();     // Clear + remove implementation
mockFn.mockRestore();   // Clear + remove + restore original (for jest.spyOn)
```

### jest.spyOn() — mock methods on real objects

```typescript
// jest.spyOn(): wrap an existing method in a spy
// Useful when you want to mock ONE method but keep others real

describe('AuthService', () => {
  it('should hash password before saving', async () => {
    // Spy on bcrypt.hash without mocking the entire module
    const bcryptHashSpy = jest.spyOn(bcrypt, 'hash')
      .mockResolvedValue('hashed_password' as never);
      // .mockResolvedValue: replace real hash with instant mock

    const result = await authService.register({
      email: 'test@test.com',
      password: 'plaintext',
    });

    expect(bcryptHashSpy).toHaveBeenCalledWith('plaintext', 12);
    // Verify bcrypt.hash was called with correct args

    expect(result.user.password).not.toBe('plaintext');
    // Password was transformed

    bcryptHashSpy.mockRestore();
    // Restore original bcrypt.hash for other tests
  });
});

// Spy on a service method:
it('should call findOne in update', async () => {
  const findOneSpy = jest.spyOn(service, 'findOne')
    .mockResolvedValue({ id: 1, email: 'test@test.com' } as User);

  await service.update(1, { firstName: 'New' });

  expect(findOneSpy).toHaveBeenCalledWith(1);
  // Verifies internal method was called correctly
});
```

### Mocking entire modules

```typescript
// Mock an entire npm module
jest.mock('bcrypt', () => ({
  hash: jest.fn().mockResolvedValue('mocked_hash'),
  compare: jest.fn().mockResolvedValue(true),
  genSalt: jest.fn().mockResolvedValue('mocked_salt'),
}));
// This replaces 'bcrypt' for the entire test file
// Every import of 'bcrypt' in tested files gets the mock

// Mock NestJS JwtService
const mockJwtService = {
  sign: jest.fn().mockReturnValue('mock.jwt.token'),
  verify: jest.fn().mockReturnValue({ sub: '1', email: 'test@test.com' }),
  verifyAsync: jest.fn().mockResolvedValue({ sub: '1', email: 'test@test.com' }),
  signAsync: jest.fn().mockResolvedValue('mock.jwt.token'),
};

// Mock ConfigService
const mockConfigService = {
  get: jest.fn((key: string) => {
    const config: Record<string, string> = {
      'JWT_SECRET': 'test-secret',
      'JWT_EXPIRES_IN': '15m',
      'PORT': '3000',
    };
    return config[key];
  }),
  getOrThrow: jest.fn((key: string) => {
    const value = mockConfigService.get(key);
    if (!value) throw new Error(`Config key not found: ${key}`);
    return value;
  }),
};

// Use in TestingModule:
{
  provide: JwtService,
  useValue: mockJwtService,
},
{
  provide: ConfigService,
  useValue: mockConfigService,
},
```

---

## 5. Testing Controllers

```typescript
// src/users/users.controller.spec.ts

import { Test, TestingModule } from '@nestjs/testing';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { NotFoundException } from '@nestjs/common';

// Mock the UsersService — controller tests should not test service logic
const mockUsersService = {
  findAll: jest.fn(),
  findOne: jest.fn(),
  create: jest.fn(),
  update: jest.fn(),
  remove: jest.fn(),
};

describe('UsersController', () => {
  let controller: UsersController;
  let usersService: jest.Mocked<UsersService>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [
        {
          provide: UsersService,
          useValue: mockUsersService,
        },
      ],
    }).compile();

    controller = module.get<UsersController>(UsersController);
    usersService = module.get(UsersService);
    // module.get() returns the mock since we provided mockUsersService
  });

  afterEach(() => {
    jest.clearAllMocks();
    // Reset all mocks after each test — fresh state
  });

  // ── Controller is defined ───────────────────────────────────────────────
  it('should be defined', () => {
    expect(controller).toBeDefined();
  });

  // ── findAll ─────────────────────────────────────────────────────────────
  describe('findAll()', () => {
    it('should call service.findAll and return result', async () => {
      const expectedResult = {
        data: [{ id: 1, email: 'alice@test.com' }],
        meta: { total: 1, page: 1, limit: 20, totalPages: 1 },
      };

      mockUsersService.findAll.mockResolvedValue(expectedResult);

      const result = await controller.findAll({ page: 1, limit: 20 });

      expect(result).toEqual(expectedResult);
      expect(mockUsersService.findAll).toHaveBeenCalledWith({ page: 1, limit: 20 });
    });
  });

  // ── findOne ─────────────────────────────────────────────────────────────
  describe('findOne()', () => {
    it('should return a user by id', async () => {
      const user = { id: 1, email: 'alice@test.com' };
      mockUsersService.findOne.mockResolvedValue(user);

      const result = await controller.findOne(1);
      // Note: ParseIntPipe runs during HTTP requests, not in unit tests
      // So we pass the number directly (already parsed)

      expect(result).toEqual(user);
      expect(mockUsersService.findOne).toHaveBeenCalledWith(1);
    });

    it('should propagate NotFoundException from service', async () => {
      mockUsersService.findOne.mockRejectedValue(
        new NotFoundException('User #999 not found'),
      );

      await expect(controller.findOne(999)).rejects.toThrow(NotFoundException);
    });
  });

  // ── create ──────────────────────────────────────────────────────────────
  describe('create()', () => {
    it('should create a new user', async () => {
      const createDto: CreateUserDto = {
        firstName: 'Alice',
        lastName: 'Smith',
        email: 'alice@test.com',
        password: 'SecurePass@123',
      };

      const createdUser = { id: 1, ...createDto };
      mockUsersService.create.mockResolvedValue(createdUser);

      const result = await controller.create(createDto);

      expect(result).toEqual(createdUser);
      expect(mockUsersService.create).toHaveBeenCalledWith(createDto);
    });
  });

  // ── update ──────────────────────────────────────────────────────────────
  describe('update()', () => {
    it('should update a user', async () => {
      const updateDto = { firstName: 'Alicia' };
      const updatedUser = { id: 1, firstName: 'Alicia', email: 'alice@test.com' };

      mockUsersService.update.mockResolvedValue(updatedUser);

      const result = await controller.update(1, updateDto);

      expect(result).toEqual(updatedUser);
      expect(mockUsersService.update).toHaveBeenCalledWith(1, updateDto);
    });
  });

  // ── remove ──────────────────────────────────────────────────────────────
  describe('remove()', () => {
    it('should call service.remove with correct id', async () => {
      mockUsersService.remove.mockResolvedValue(undefined);

      await controller.remove(1);

      expect(mockUsersService.remove).toHaveBeenCalledWith(1);
      expect(mockUsersService.remove).toHaveBeenCalledTimes(1);
    });
  });
});
```

---

## 6. Testing Guards, Pipes, and Interceptors

### Testing a Guard

```typescript
// src/auth/guards/roles.guard.spec.ts

import { Test } from '@nestjs/testing';
import { ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { RolesGuard } from './roles.guard';
import { Role } from '../../common/enums/role.enum';

describe('RolesGuard', () => {
  let guard: RolesGuard;
  let reflector: Reflector;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        RolesGuard,
        {
          provide: Reflector,
          useValue: {
            getAllAndOverride: jest.fn(),
          },
        },
      ],
    }).compile();

    guard = module.get<RolesGuard>(RolesGuard);
    reflector = module.get<Reflector>(Reflector);
  });

  // Helper: create a mock ExecutionContext
  const createMockContext = (user: any, handler = {}, cls = {}): ExecutionContext => ({
    getHandler: () => handler,
    getClass: () => cls,
    switchToHttp: () => ({
      getRequest: () => ({ user }),
    }),
  } as any);

  it('should allow access when no roles are required', () => {
    jest.spyOn(reflector, 'getAllAndOverride').mockReturnValue(null);
    // No @Roles() decorator → no required roles

    const context = createMockContext({ role: Role.USER });
    const result = guard.canActivate(context);

    expect(result).toBe(true);
  });

  it('should allow access when user has required role', () => {
    jest.spyOn(reflector, 'getAllAndOverride').mockReturnValue([Role.ADMIN]);
    // @Roles(Role.ADMIN) is set

    const context = createMockContext({ role: Role.ADMIN });
    const result = guard.canActivate(context);

    expect(result).toBe(true);
  });

  it('should deny access when user lacks required role', () => {
    jest.spyOn(reflector, 'getAllAndOverride').mockReturnValue([Role.ADMIN]);

    const context = createMockContext({ role: Role.USER });

    expect(() => guard.canActivate(context)).toThrow(ForbiddenException);
  });

  it('should allow access when user has one of multiple required roles', () => {
    jest.spyOn(reflector, 'getAllAndOverride')
      .mockReturnValue([Role.ADMIN, Role.MODERATOR]);

    const context = createMockContext({ role: Role.MODERATOR });
    const result = guard.canActivate(context);

    expect(result).toBe(true);
  });
});
```

### Testing a Pipe

```typescript
// src/common/pipes/parse-sort.pipe.spec.ts

import { ParseSortPipe } from './parse-sort.pipe';
import { BadRequestException } from '@nestjs/common';

describe('ParseSortPipe', () => {
  let pipe: ParseSortPipe;

  beforeEach(() => {
    pipe = new ParseSortPipe();
    // Pipes with no dependencies can be instantiated directly
    // No TestingModule needed
  });

  it('should parse a valid sort string', () => {
    const result = pipe.transform('name:asc,createdAt:desc', {} as any);

    expect(result).toEqual([
      { field: 'name', order: 'ASC' },
      { field: 'createdAt', order: 'DESC' },
    ]);
  });

  it('should default to DESC when order not specified', () => {
    const result = pipe.transform('name', {} as any);
    expect(result).toEqual([{ field: 'name', order: 'DESC' }]);
  });

  it('should return default sort when no value provided', () => {
    const result = pipe.transform(undefined, {} as any);
    expect(result).toEqual([{ field: 'createdAt', order: 'DESC' }]);
  });

  it('should throw BadRequestException for invalid field', () => {
    expect(() => pipe.transform('invalidField:asc', {} as any))
      .toThrow(BadRequestException);
  });

  it('should throw BadRequestException for invalid order', () => {
    expect(() => pipe.transform('name:invalid', {} as any))
      .toThrow(BadRequestException);
  });
});
```

### Testing an Interceptor

```typescript
// src/common/interceptors/transform.interceptor.spec.ts

import { TransformInterceptor } from './transform.interceptor';
import { ExecutionContext, CallHandler } from '@nestjs/common';
import { of } from 'rxjs';

describe('TransformInterceptor', () => {
  let interceptor: TransformInterceptor<any>;

  beforeEach(() => {
    interceptor = new TransformInterceptor();
  });

  const createMockContext = (statusCode = 200) => ({
    switchToHttp: () => ({
      getRequest: () => ({ url: '/api/v1/users' }),
      getResponse: () => ({ statusCode }),
    }),
  } as ExecutionContext);

  it('should wrap response in standard envelope', (done) => {
    const data = { id: 1, name: 'Alice' };
    const mockCallHandler: CallHandler = {
      handle: () => of(data),
      // of(data): creates an Observable that immediately emits data
    };

    const context = createMockContext(200);
    const result$ = interceptor.intercept(context, mockCallHandler);

    result$.subscribe({
      next: (value) => {
        expect(value.success).toBe(true);
        expect(value.data).toEqual(data);
        expect(value.statusCode).toBe(200);
        expect(value.timestamp).toBeDefined();
        expect(value.path).toBe('/api/v1/users');
        done();
      },
      error: done,
    });
  });

  it('should handle paginated response', (done) => {
    const paginatedData = {
      data: [{ id: 1 }],
      meta: { total: 1, page: 1 },
    };

    const mockCallHandler: CallHandler = {
      handle: () => of(paginatedData),
    };

    interceptor.intercept(createMockContext(), mockCallHandler).subscribe({
      next: (value) => {
        expect(value.data).toEqual(paginatedData.data);
        expect(value.meta).toEqual(paginatedData.meta);
        done();
      },
      error: done,
    });
  });
});
```

---

## 7. Integration Tests — With Real Database

```typescript
// src/users/users.service.integration.spec.ts
// Uses real TypeORM with SQLite in-memory database
// Tests actual SQL queries and ORM behavior

import { Test, TestingModule } from '@nestjs/testing';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';

describe('UsersService (Integration)', () => {
  let service: UsersService;
  let repository: Repository<User>;
  let module: TestingModule;

  beforeAll(async () => {
    // beforeAll for expensive setup (DB connection)
    module = await Test.createTestingModule({
      imports: [
        TypeOrmModule.forRoot({
          type: 'sqlite',
          database: ':memory:',
          // ':memory:': SQLite in-memory DB — fast, no file, auto-cleaned
          entities: [User],
          synchronize: true,
          // synchronize: true is fine for tests — it's an in-memory DB
          dropSchema: true,
          // dropSchema: wipe DB before starting (clean slate for tests)
          logging: false,
          // logging: false — don't spam SQL in test output
        }),
        TypeOrmModule.forFeature([User]),
      ],
      providers: [UsersService],
    }).compile();

    service = module.get<UsersService>(UsersService);
    repository = module.get<Repository<User>>(getRepositoryToken(User));
  });

  afterAll(async () => {
    await module.close();
    // Close the DB connection after all tests complete
  });

  beforeEach(async () => {
    // Clean the database before each test for isolation
    await repository.clear();
    // repository.clear(): DELETE FROM users (fast, no reset IDs)
    // Alternative: await repository.delete({}) — same effect
  });

  it('should create and find a user', async () => {
    // This tests REAL SQL INSERT + SELECT
    const created = await service.create({
      firstName: 'Alice',
      lastName: 'Smith',
      email: 'alice@test.com',
      password: 'Test@1234',
    });

    expect(created.id).toBeDefined();
    // ID should be auto-generated by SQLite

    const found = await service.findOne(created.id);
    expect(found.email).toBe('alice@test.com');
    expect(found.firstName).toBe('Alice');
  });

  it('should enforce unique email constraint', async () => {
    await service.create({
      firstName: 'Alice',
      lastName: 'Smith',
      email: 'alice@test.com',
      password: 'Test@1234',
    });

    // Second user with same email — should throw ConflictException
    await expect(
      service.create({
        firstName: 'Alice 2',
        lastName: 'Smith',
        email: 'alice@test.com',
        password: 'Test@1234',
      }),
    ).rejects.toThrow(ConflictException);
  });

  it('should paginate results correctly', async () => {
    // Create 5 users
    for (let i = 1; i <= 5; i++) {
      await repository.save(
        repository.create({
          firstName: `User${i}`,
          lastName: 'Test',
          email: `user${i}@test.com`,
          password: 'hashed',
        }),
      );
    }

    const page1 = await service.findAll({ page: 1, limit: 2 });
    expect(page1.data).toHaveLength(2);
    expect(page1.meta.total).toBe(5);
    expect(page1.meta.totalPages).toBe(3);

    const page3 = await service.findAll({ page: 3, limit: 2 });
    expect(page3.data).toHaveLength(1); // Last page has 1 item
  });
});
```

---

## 8. E2E Tests — Full Stack with Supertest

### E2E test setup

```typescript
// test/users.e2e-spec.ts
// Tests the entire stack: HTTP → Controller → Service → Database

import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
// supertest: HTTP testing library — makes real HTTP requests to your app
import { AppModule } from '../src/app.module';
import { TypeOrmModule } from '@nestjs/typeorm';
import { DataSource } from 'typeorm';

describe('Users (E2E)', () => {
  let app: INestApplication;
  let dataSource: DataSource;
  let authToken: string;  // JWT token for authenticated requests

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [
        // Override database config for testing
        AppModule,
        // If AppModule uses forRoot (not forRootAsync), you might need to
        // override it here
      ],
    })
    .overrideProvider(TypeOrmModule)  // Override if needed
    .useValue(/* test DB config */)
    .compile();

    app = moduleFixture.createNestApplication();
    // createNestApplication: creates a full NestJS app instance
    // (with HTTP server, but not listening yet)

    // Apply the SAME middleware as production
    app.setGlobalPrefix('api/v1');
    app.useGlobalPipes(new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }));

    await app.init();
    // Start the application (initializes modules but doesn't listen on port)
    // Supertest handles the HTTP without needing a real port

    dataSource = app.get(DataSource);
  });

  afterAll(async () => {
    await dataSource.dropDatabase();
    // Wipe test database after all tests
    await app.close();
    // Close the app and all connections
  });

  beforeEach(async () => {
    // Clear all tables before each test
    const entities = dataSource.entityMetadatas;
    for (const entity of entities) {
      const repository = dataSource.getRepository(entity.name);
      await repository.clear();
    }
  });

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // AUTH FLOW
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  describe('/auth', () => {
    it('POST /auth/register → 201 with tokens', async () => {
      const response = await request(app.getHttpServer())
        // app.getHttpServer(): gets the underlying HTTP server
        // supertest wraps it without starting on a port
        .post('/api/v1/auth/register')
        .send({
          firstName: 'Alice',
          lastName: 'Smith',
          email: 'alice@test.com',
          password: 'SecurePass@123',
        })
        .expect(201);
        // .expect(): assert the HTTP status code

      expect(response.body.data.accessToken).toBeDefined();
      expect(response.body.data.refreshToken).toBeDefined();
      expect(response.body.data.user.email).toBe('alice@test.com');
      expect(response.body.data.user.password).toBeUndefined();
      // Password should NEVER be in response

      // Save token for later tests
      authToken = response.body.data.accessToken;
    });

    it('POST /auth/register → 400 for invalid email', async () => {
      await request(app.getHttpServer())
        .post('/api/v1/auth/register')
        .send({
          firstName: 'Bob',
          lastName: 'Jones',
          email: 'not-an-email',
          password: 'SecurePass@123',
        })
        .expect(400);
        // ValidationPipe rejects invalid email
    });

    it('POST /auth/register → 409 for duplicate email', async () => {
      // Register first
      await request(app.getHttpServer())
        .post('/api/v1/auth/register')
        .send({
          firstName: 'Alice',
          lastName: 'Smith',
          email: 'alice@test.com',
          password: 'SecurePass@123',
        });

      // Try to register again with same email
      await request(app.getHttpServer())
        .post('/api/v1/auth/register')
        .send({
          firstName: 'Alice',
          lastName: 'Smith',
          email: 'alice@test.com',
          password: 'SecurePass@123',
        })
        .expect(409);
    });

    it('POST /auth/login → 200 with tokens', async () => {
      // Register first
      await request(app.getHttpServer())
        .post('/api/v1/auth/register')
        .send({
          firstName: 'Alice',
          email: 'alice@test.com',
          password: 'SecurePass@123',
        });

      // Then login
      const response = await request(app.getHttpServer())
        .post('/api/v1/auth/login')
        .send({ email: 'alice@test.com', password: 'SecurePass@123' })
        .expect(200);

      expect(response.body.data.accessToken).toBeDefined();
    });

    it('POST /auth/login → 401 for wrong password', async () => {
      await request(app.getHttpServer())
        .post('/api/v1/auth/login')
        .send({ email: 'alice@test.com', password: 'wrongpassword' })
        .expect(401);
    });
  });

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // USERS — AUTHENTICATED ROUTES
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  describe('/users', () => {
    // Register and get token before each test
    beforeEach(async () => {
      const registerResponse = await request(app.getHttpServer())
        .post('/api/v1/auth/register')
        .send({
          firstName: 'Alice',
          lastName: 'Smith',
          email: 'alice@test.com',
          password: 'SecurePass@123',
        });

      authToken = registerResponse.body.data.accessToken;
    });

    it('GET /users/me → 200 with user profile', async () => {
      const response = await request(app.getHttpServer())
        .get('/api/v1/auth/me')
        .set('Authorization', `Bearer ${authToken}`)
        // .set(): adds HTTP headers
        .expect(200);

      expect(response.body.data.email).toBe('alice@test.com');
      expect(response.body.data.password).toBeUndefined();
    });

    it('GET /users/me → 401 without token', async () => {
      await request(app.getHttpServer())
        .get('/api/v1/auth/me')
        // No Authorization header
        .expect(401);
    });

    it('GET /users/me → 401 with invalid token', async () => {
      await request(app.getHttpServer())
        .get('/api/v1/auth/me')
        .set('Authorization', 'Bearer invalid.token.here')
        .expect(401);
    });

    it('PATCH /users/1 → 200 updates user', async () => {
      const meResponse = await request(app.getHttpServer())
        .get('/api/v1/auth/me')
        .set('Authorization', `Bearer ${authToken}`);

      const userId = meResponse.body.data.id;

      const response = await request(app.getHttpServer())
        .patch(`/api/v1/users/${userId}`)
        .set('Authorization', `Bearer ${authToken}`)
        .send({ firstName: 'Alicia' })
        .expect(200);

      expect(response.body.data.firstName).toBe('Alicia');
    });
  });

  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  // COMPLETE FLOW TEST
  // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  describe('Complete user journey', () => {
    it('should register → login → get profile → update profile', async () => {
      // Step 1: Register
      const registerRes = await request(app.getHttpServer())
        .post('/api/v1/auth/register')
        .send({
          firstName: 'Bob',
          lastName: 'Jones',
          email: 'bob@test.com',
          password: 'SecurePass@123',
        })
        .expect(201);

      const { accessToken } = registerRes.body.data;
      const userId = registerRes.body.data.user.id;

      // Step 2: Get profile
      const profileRes = await request(app.getHttpServer())
        .get('/api/v1/auth/me')
        .set('Authorization', `Bearer ${accessToken}`)
        .expect(200);

      expect(profileRes.body.data.email).toBe('bob@test.com');

      // Step 3: Update profile
      const updateRes = await request(app.getHttpServer())
        .patch(`/api/v1/users/${userId}`)
        .set('Authorization', `Bearer ${accessToken}`)
        .send({ firstName: 'Bobby' })
        .expect(200);

      expect(updateRes.body.data.firstName).toBe('Bobby');

      // Step 4: Verify update persisted
      const finalProfile = await request(app.getHttpServer())
        .get('/api/v1/auth/me')
        .set('Authorization', `Bearer ${accessToken}`)
        .expect(200);

      expect(finalProfile.body.data.firstName).toBe('Bobby');
    });
  });
});
```

---

## 9. Testing Auth — JWT and Guards

```typescript
// src/auth/auth.service.spec.ts

import { Test, TestingModule } from '@nestjs/testing';
import { JwtService } from '@nestjs/jwt';
import { getRepositoryToken } from '@nestjs/typeorm';
import { UnauthorizedException, ConflictException } from '@nestjs/common';
import * as bcrypt from 'bcrypt';
import { AuthService } from './auth.service';
import { User } from '../users/entities/user.entity';

describe('AuthService', () => {
  let authService: AuthService;
  let jwtService: jest.Mocked<JwtService>;

  const mockUserRepository = {
    findOne: jest.fn(),
    create: jest.fn(),
    save: jest.fn(),
    update: jest.fn(),
    createQueryBuilder: jest.fn(() => ({
      where: jest.fn().mockReturnThis(),
      addSelect: jest.fn().mockReturnThis(),
      getOne: jest.fn(),
    })),
  };

  const mockJwtService = {
    sign: jest.fn().mockReturnValue('mock.access.token'),
    verifyAsync: jest.fn(),
  };

  const mockConfigService = {
    getOrThrow: jest.fn((key: string) => {
      const configs: Record<string, string> = {
        JWT_SECRET: 'test-secret',
        JWT_EXPIRES_IN: '15m',
        JWT_REFRESH_SECRET: 'test-refresh-secret',
        JWT_REFRESH_EXPIRES_IN: '7d',
      };
      return configs[key] || 'test-value';
    }),
    get: jest.fn((key: string, def?: string) => {
      return mockConfigService.getOrThrow(key) || def;
    }),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        AuthService,
        { provide: getRepositoryToken(User), useValue: mockUserRepository },
        { provide: JwtService, useValue: mockJwtService },
        { provide: ConfigService, useValue: mockConfigService },
      ],
    }).compile();

    authService = module.get<AuthService>(AuthService);
  });

  afterEach(() => jest.clearAllMocks());

  describe('validateUser()', () => {
    const mockUser = {
      id: 1,
      email: 'alice@test.com',
      password: '$2b$12$hashedpassword',
      isActive: true,
    } as User;

    it('should return user without password when credentials are valid', async () => {
      // Mock the QueryBuilder chain
      const getOneMock = jest.fn().mockResolvedValue(mockUser);
      mockUserRepository.createQueryBuilder.mockReturnValue({
        where: jest.fn().mockReturnThis(),
        addSelect: jest.fn().mockReturnThis(),
        getOne: getOneMock,
      });

      // Mock bcrypt to avoid actual hashing in tests
      jest.spyOn(bcrypt, 'compare').mockResolvedValue(true as never);

      const result = await authService.validateUser(
        'alice@test.com',
        'correctpassword',
      );

      expect(result).toBeDefined();
      expect(result?.password).toBeUndefined();
      // Password must not be returned
    });

    it('should return null for invalid password', async () => {
      const getOneMock = jest.fn().mockResolvedValue(mockUser);
      mockUserRepository.createQueryBuilder.mockReturnValue({
        where: jest.fn().mockReturnThis(),
        addSelect: jest.fn().mockReturnThis(),
        getOne: getOneMock,
      });

      jest.spyOn(bcrypt, 'compare').mockResolvedValue(false as never);

      const result = await authService.validateUser(
        'alice@test.com',
        'wrongpassword',
      );

      expect(result).toBeNull();
    });

    it('should return null and still run bcrypt for non-existent email (timing attack prevention)', async () => {
      const getOneMock = jest.fn().mockResolvedValue(null);
      mockUserRepository.createQueryBuilder.mockReturnValue({
        where: jest.fn().mockReturnThis(),
        addSelect: jest.fn().mockReturnThis(),
        getOne: getOneMock,
      });

      const compareSpy = jest.spyOn(bcrypt, 'compare').mockResolvedValue(false as never);

      const result = await authService.validateUser(
        'notexist@test.com',
        'password',
      );

      expect(result).toBeNull();
      expect(compareSpy).toHaveBeenCalled();
      // bcrypt.compare MUST still be called even when user not found
      // Prevents timing attacks
    });
  });

  describe('login()', () => {
    it('should return access and refresh tokens', async () => {
      const user = { id: 1, email: 'alice@test.com', role: 'user' } as User;

      mockUserRepository.update.mockResolvedValue({});

      jest.spyOn(bcrypt, 'hash').mockResolvedValue('hashed_refresh' as never);

      const result = await authService.login(user);

      expect(result.accessToken).toBeDefined();
      expect(result.refreshToken).toBeDefined();
      expect(result.user.email).toBe('alice@test.com');
      expect(mockJwtService.sign).toHaveBeenCalledTimes(2);
      // Called twice: once for access token, once for refresh token
    });
  });
});
```

---

## 10. Code Coverage — What It Means

```bash
# Generate coverage report
npm run test:cov

# Output (example):
# ─────────────────────────────────────────────────────────────
# File                  | % Stmts | % Branch | % Funcs | % Lines
# ─────────────────────────────────────────────────────────────
# users.service.ts      |   95.23 |    88.00 |  100.00 |   95.23
# users.controller.ts   |  100.00 |   100.00 |  100.00 |  100.00
# auth.service.ts       |   78.26 |    72.50 |   90.00 |   78.26
# ─────────────────────────────────────────────────────────────
```

### Coverage metrics explained

```
Statements:  How many lines/statements were executed?
  stmt: const x = 5; → 1 statement

Branches:    How many if/else paths were tested?
  if (user) { ... } else { ... }
  → 2 branches: user exists AND user null
  50% branch coverage if only one path tested

Functions:   How many functions were called?
  100%: every function in the file was called at least once

Lines:       Similar to statements (grouped by line number)
```

### Configuring coverage thresholds

```json
// jest.config.js or package.json
{
  "jest": {
    "coverageThreshold": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      },
      "./src/services/": {
        "branches": 90,
        "functions": 95,
        "lines": 90,
        "statements": 90
      }
    },
    "collectCoverageFrom": [
      "src/**/*.ts",
      "!src/**/*.spec.ts",
      "!src/**/*.e2e-spec.ts",
      "!src/**/index.ts",
      "!src/main.ts",
      "!src/**/*.module.ts",
      "!src/**/*.dto.ts",
      "!src/**/*.entity.ts"
    ]
  }
}
```

---

## 11. Testing Best Practices

```typescript
// ── 1. ARRANGE - ACT - ASSERT (AAA) ──────────────────────────────────────

it('should follow AAA pattern', async () => {
  // ARRANGE: set up state and mocks
  const userId = 1;
  const mockUser = { id: userId, email: 'test@test.com' } as User;
  userRepository.findOne.mockResolvedValue(mockUser);

  // ACT: call the thing you're testing
  const result = await service.findOne(userId);

  // ASSERT: verify expectations
  expect(result).toEqual(mockUser);
  expect(userRepository.findOne).toHaveBeenCalledWith({ where: { id: userId } });
});

// ── 2. ONE ASSERTION CONCEPT PER TEST ────────────────────────────────────

// ❌ Too many concepts in one test
it('findAll works', async () => {
  userRepository.findAndCount.mockResolvedValue([users, 10]);
  const result = await service.findAll({ page: 1, limit: 5 });
  expect(result.data).toEqual(users);
  expect(result.meta.total).toBe(10);
  expect(result.meta.totalPages).toBe(2);
  expect(userRepository.findAndCount).toHaveBeenCalled();
  // Tests 4 different things — if it fails, which one is the issue?
});

// ✅ Focused tests
it('should return data array', async () => {
  userRepository.findAndCount.mockResolvedValue([users, 10]);
  const result = await service.findAll({ page: 1, limit: 5 });
  expect(result.data).toEqual(users);
});

it('should return correct total in meta', async () => {
  userRepository.findAndCount.mockResolvedValue([users, 10]);
  const result = await service.findAll({ page: 1, limit: 5 });
  expect(result.meta.total).toBe(10);
  expect(result.meta.totalPages).toBe(2);
});

// ── 3. TEST BEHAVIOR NOT IMPLEMENTATION ─────────────────────────────────

// ❌ Testing implementation (brittle — breaks if you rename internal variables)
it('should set this.cachedUser', () => {
  service.findOne(1);
  expect(service['cachedUser']).toBeDefined(); // Testing private state
});

// ✅ Testing behavior (stable — doesn't break on refactoring)
it('should return same user on repeated calls', async () => {
  userRepository.findOne.mockResolvedValue(mockUser);
  await service.findOne(1);
  await service.findOne(1);
  expect(userRepository.findOne).toHaveBeenCalledTimes(1);
  // Only one DB call means caching is working — tests behavior
});

// ── 4. DESCRIPTIVE TEST NAMES ────────────────────────────────────────────

// ❌ Vague
it('test 1', () => {});
it('works', () => {});
it('user test', () => {});

// ✅ Self-documenting
it('should return user when valid id is provided', () => {});
it('should throw NotFoundException when user does not exist', () => {});
it('should not save password as plain text', () => {});

// ── 5. DON'T OVER-MOCK ───────────────────────────────────────────────────

// ❌ Mocking the thing you're testing
it('should create user', async () => {
  jest.spyOn(service, 'create').mockResolvedValue(mockUser);
  // This test tests nothing — you're mocking the actual method
  const result = await service.create(dto);
  expect(result).toEqual(mockUser); // Always true regardless of implementation
});

// ✅ Mock DEPENDENCIES of what you're testing, not the thing itself
it('should create user', async () => {
  userRepository.findOne.mockResolvedValue(null);     // ← mock dependency
  userRepository.create.mockReturnValue(mockUser);    // ← mock dependency
  userRepository.save.mockResolvedValue(mockUser);    // ← mock dependency
  const result = await service.create(dto);            // ← test real service
  expect(result).toEqual(mockUser);
});
```

---

## 12. Interview Questions & Answers

**Q1: What is the difference between unit tests, integration tests, and e2e tests in NestJS?**

> "Unit tests test a single class in complete isolation — all dependencies are mocked. A service unit test mocks the repository, JWT service, and config, then tests only the service's logic. They run in milliseconds and you should have many of them. Integration tests test a group of components working together with real infrastructure — typically a service with a real database (SQLite in-memory). They verify that the SQL queries work correctly and ORM behavior is right, not just that the logic is correct. E2E tests test the entire HTTP stack — they make real HTTP requests to a running NestJS app and verify the response. They test your complete user flows including middleware, guards, pipes, and serialization. E2E tests are slowest but give the most confidence. The testing pyramid says: many unit, some integration, few e2e."

---

**Q2: How do you mock a TypeORM repository in NestJS tests?**

> "TypeORM repositories are injected using the token returned by `getRepositoryToken(Entity)`. In the test module, I provide a mock object with the same interface using `{ provide: getRepositoryToken(User), useValue: mockRepository }`. The mock is a plain object with jest.fn() for each method I need: `{ find: jest.fn(), findOne: jest.fn(), save: jest.fn(), create: jest.fn() }`. I use a factory function `useFactory: mockUserRepository` rather than `useValue: mockObject` to get a fresh mock per test, preventing state leaking between tests. For QueryBuilder, I mock `createQueryBuilder` to return an object with fluent methods that each return `this` for chaining. In each test, I configure what the mock returns using `mockResolvedValue` or `mockRejectedValue`."

---

**Q3: What is TestingModule and how does it differ from a real NestJS module?**

> "TestingModule is created with `Test.createTestingModule()` — it's an isolated NestJS DI container for testing. It works identically to a real module: you declare providers, imports, and controllers the same way. The differences are: no HTTP server is started, you can override any provider with mocks using `useValue` or `overrideProvider().useValue()`, and it compiles synchronously enough to be used in tests. You call `.compile()` to finalize it, then `module.get()` to retrieve any provider. For e2e tests, you additionally call `moduleFixture.createNestApplication()` to create a real app instance and `app.init()` to initialize it — Supertest then makes HTTP requests directly to the app server object without needing an actual port."

---

**Q4: How do you test async code that throws exceptions?**

> "Jest provides `.rejects` for testing rejected promises. The pattern is `await expect(asyncFn()).rejects.toThrow(SomeException)`. You can also check the message: `.rejects.toThrow('Specific message')`, or check the object shape: `.rejects.toMatchObject({ message: 'User not found', status: 404 })`. For synchronous throws, use `expect(() => syncFn()).toThrow(Error)` — note the function wrapper, because if you call `syncFn()` directly without the wrapper, it throws before `expect` can catch it. A common mistake is forgetting to `await` the assertion for async functions — `expect(asyncFn()).rejects.toThrow()` without await will not actually assert anything; the test passes even if the assertion would fail."

---

**Q5: What should you NOT test?**

> "I don't test third-party library internals — TypeORM, Passport, NestJS core behavior. I don't test TypeScript type checking — if TypeScript compiles, the types are correct. I don't test simple DTOs with only class-validator decorators — these are implicitly tested by e2e tests. I avoid testing obvious getter/setter boilerplate unless it has logic. I don't write tests for test quality metrics — 100% coverage is not the goal. High-value tests cover: complex business logic with multiple conditions, error cases that are easy to miss, security-critical paths (auth, authorization, input sanitization), and integration between components where bugs commonly live. Tests should give you confidence to refactor, not just push coverage numbers up."

---

**Q6: How do you handle database isolation in tests?**

> "For unit tests: mock the repository — no real database, perfect isolation, fast. For integration tests with a real database: use SQLite in-memory (`database: ':memory:'`) with `synchronize: true` and `dropSchema: true` — creates a fresh schema on every test run. Clear data between tests with `repository.clear()` in `beforeEach`. For e2e tests: either use the same SQLite in-memory approach or use a real PostgreSQL test database (separate from dev/production) — I prefer test-specific PostgreSQL because it tests the actual SQL dialect. I use `beforeAll` to set up the module once, `afterAll` to close it, and `beforeEach` to clear tables. Transactions are another option: wrap each test in a transaction and roll it back after — no cleanup needed."

---

## Quick Reference — Day 13 Cheat Sheet

```
Test file naming:
  users.service.spec.ts     → unit test
  users.e2e-spec.ts         → e2e test
  jest finds files ending in .spec.ts automatically

Jest structure:
  describe('Group')          → test suite
  it('behavior') / test()    → individual test
  beforeAll / afterAll       → once per describe
  beforeEach / afterEach     → before/after each test

TestingModule:
  Test.createTestingModule({ providers, imports, controllers })
  .compile()                 → build the module
  module.get<T>(Token)       → get a provider

Mock patterns:
  jest.fn()                                → empty mock function
  jest.fn().mockReturnValue(x)             → sync return
  jest.fn().mockResolvedValue(x)           → async success
  jest.fn().mockRejectedValue(new Error()) → async failure
  jest.fn().mockReturnValueOnce(x)         → first call only
  jest.spyOn(obj, 'method').mockReturnValue(x)

TypeORM mock:
  { provide: getRepositoryToken(User), useFactory: mockRepository }
  mockRepository = () => ({ find: jest.fn(), findOne: jest.fn(), ... })

Key assertions:
  expect(x).toBe(y)           → strict equality (primitives)
  expect(x).toEqual(y)        → deep equality (objects)
  expect(x).toMatchObject(y)  → partial match
  expect(x).toBeDefined()     → not undefined
  expect(fn).toHaveBeenCalledWith(args)
  await expect(asyncFn()).rejects.toThrow(Exception)
  await expect(asyncFn()).resolves.toEqual(value)
  expect(fn).not.toHaveBeenCalled()

E2E with Supertest:
  request(app.getHttpServer())
    .post('/api/v1/users')
    .set('Authorization', `Bearer ${token}`)
    .send({ ... })
    .expect(201)
    .then(res => expect(res.body).toMatchObject({...}))

Coverage:
  npm run test:cov
  Aim for 80%+ on business logic
  Don't aim for 100% — test behavior, not coverage numbers

What to test:
  ✓ All service methods (success + failure paths)
  ✓ Guard canActivate logic
  ✓ Error handling and exception types
  ✓ Complete e2e flows (register → login → use protected route)
  ✗ Third-party library internals
  ✗ TypeScript types
  ✗ Simple CRUD with no logic
```

---

*Day 13 complete. Tomorrow — Day 14: Performance, Caching & Queues — Redis caching with CacheManager, Bull job queues, Bull dashboard, compression, throttling, and PM2 clustering.*
