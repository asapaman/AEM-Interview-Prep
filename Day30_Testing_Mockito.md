# Day 30 — Unit Testing (JUnit 5 & Mockito)
**Level:** Intermediate | **Java Version:** 8+

---

## 🌟 Why Write Unit Tests?

Unit tests verify that a specific, isolated "unit" of code (usually a single method or class) works exactly as intended.
- **Safety Net:** Prevents regressions (breaking existing code when adding new features).
- **Design Feedback:** Hard-to-test code is usually poorly designed (tightly coupled).
- **Documentation:** Tests show exactly how a class is supposed to be used.

*Test-Driven Development (TDD) workflow:* Red (Write failing test) -> Green (Write code to pass) -> Refactor (Clean up).

---

## 🧪 JUnit 5 Basics

JUnit is the standard testing framework in Java. JUnit 5 consists of 3 sub-projects: Platform, Jupiter (the new API), and Vintage (for legacy JUnit 3/4 tests).

### The Class Under Test
```java
public class Calculator {
    public int add(int a, int b) { return a + b; }
    public int divide(int a, int b) {
        if (b == 0) throw new IllegalArgumentException("Cannot divide by zero");
        return a / b;
    }
}
```

### The Test Class
```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

// Note: Test classes do not need to be public in JUnit 5
class CalculatorTest {

    private Calculator calc;

    // Runs ONCE before all tests in this class (must be static)
    @BeforeAll
    static void setupAll() { System.out.println("Starting tests..."); }

    // Runs BEFORE EACH test method
    @BeforeEach
    void setup() { 
        calc = new Calculator(); // Fresh instance for every test
    }

    @Test
    @DisplayName("1 + 1 = 2") // Custom name for the test report
    void testAddition() {
        // Arrange (Setup inputs)
        int a = 1; int b = 1;
        // Act (Call the method)
        int result = calc.add(a, b);
        // Assert (Verify output)
        assertEquals(2, result, "1 + 1 should equal 2");
    }

    @Test
    @DisplayName("Division by Zero throws Exception")
    void testDivisionByZero() {
        // AssertThrows catches the exception and passes the test
        Exception exception = assertThrows(IllegalArgumentException.class, () -> {
            calc.divide(10, 0); // This should throw!
        });
        
        assertEquals("Cannot divide by zero", exception.getMessage());
    }

    // Runs AFTER EACH test method
    @AfterEach
    void teardown() { calc = null; }
    
    // Runs ONCE after all tests (must be static)
    @AfterAll
    static void teardownAll() { System.out.println("All tests finished."); }
}
```

---

## 🎭 Mocking with Mockito

Often, the class you want to test depends on other classes (e.g., a Service depends on a Database Repository). You don't want to connect to a real database during a unit test!

**Mockito** allows you to create "mock" (fake) objects to simulate the behavior of dependencies.

### The Code
```java
// Dependency
public interface UserRepository {
    User findById(int id);
    boolean save(User user);
}

// Class under test
public class UserService {
    private final UserRepository repository;

    // Dependency Injection makes this testable!
    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    public String getUserNameOrDefault(int id) {
        User user = repository.findById(id);
        return (user != null) ? user.getName() : "Guest";
    }
}
```

### The Test with Mocks
```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(MockitoExtension.class) // Enables Mockito annotations
class UserServiceTest {

    @Mock
    private UserRepository mockRepository; // Creates a fake UserRepository

    @InjectMocks
    private UserService userService; // Injects the mockRepository into the UserService constructor

    @Test
    void testGetUserName_UserExists() {
        // 1. Arrange: Train the mock
        User fakeUser = new User(1, "Aman");
        // "When findById(1) is called, return fakeUser"
        when(mockRepository.findById(1)).thenReturn(fakeUser);

        // 2. Act: Call the real method
        String name = userService.getUserNameOrDefault(1);

        // 3. Assert: Verify the result
        assertEquals("Aman", name);
        
        // 4. Verify: Ensure the mock was actually called
        verify(mockRepository, times(1)).findById(1);
    }

    @Test
    void testGetUserName_UserNotFound() {
        // Arrange: Train the mock to return null for any integer ID
        when(mockRepository.findById(anyInt())).thenReturn(null);

        // Act
        String name = userService.getUserNameOrDefault(999);

        // Assert
        assertEquals("Guest", name);
    }
}
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between a Unit Test and an Integration Test?**
> A **Unit Test** tests a single class or method in complete isolation. All external dependencies (DBs, networks, other classes) are mocked. It is very fast. An **Integration Test** tests how multiple components work together, often involving real databases, file systems, or Spring contexts. It is slower but catches integration bugs.

**Q2. What are the 3 A's of Unit Testing?**
> **Arrange:** Set up the test data, instantiate classes, and train mocks.
> **Act:** Call the specific method you are testing.
> **Assert:** Verify that the returned value or state matches your expectations.

**Q3. What does `@Mock` vs `@InjectMocks` do in Mockito?**
> `@Mock` creates a fake instance (a shell) of a class or interface. It doesn't execute real code; it just returns default values (null/0/false) unless trained with `when()`. `@InjectMocks` creates an instance of the *real* class being tested, and automatically passes the created `@Mock`s into its constructor (or setters).

**Q4. Can you mock private methods or static methods with Mockito?**
> Standard Mockito cannot mock `private`, `static`, or `final` methods directly (because it generates proxy subclasses). For static methods, you need `mockito-inline` (built-in for newer versions) and use `mockStatic()`. Private methods generally shouldn't be mocked; you should test them via the public methods that call them. If you absolutely must mock private/static/final, use the **PowerMock** library (though its use is heavily discouraged in modern Java).

**Q5. What is the difference between `when(...).thenReturn(...)` and `verify(...)`?**
> `when()` is used during the **Arrange** phase to *stub* a method (tell it what to return when called). `verify()` is used in the **Assert** phase to check *if* a method was called, how many times it was called, or what arguments it was called with.

---

## ✅ Best Practices

1. **One Assertion Concept per Test.** A single test method should test one specific scenario. Don't write a giant test method that tests 15 different things.
2. **Name tests clearly.** Use descriptive names. A popular format is `MethodName_StateUnderTest_ExpectedBehavior` (e.g., `divide_ByZero_ThrowsException`).
3. **Don't test framework code.** Don't write tests to verify that `ArrayList.add()` works. Focus on testing *your* business logic.
4. **Use Dependency Injection.** If a class instantiates its dependencies internally using `new`, it is almost impossible to unit test effectively. Pass dependencies through constructors.
5. **Keep tests fast.** Unit tests should run in milliseconds. Avoid database connections, file I/O, or `Thread.sleep` in unit tests.

---

## 🛠️ Hands-on Practice

1. Write a `StringManipulator` class with a method `reverse(String input)`. Handle null inputs (throw `IllegalArgumentException`). Write 3 JUnit tests for it: normal string, empty string, and null string.
2. Given an `EmailService` interface (`boolean sendEmail(String to, String body)`) and an `OrderProcessor` class that depends on it. Write a test for `OrderProcessor.processOrder()` using Mockito to mock the `EmailService`. Verify that `sendEmail` is called exactly once when the order is processed successfully.
