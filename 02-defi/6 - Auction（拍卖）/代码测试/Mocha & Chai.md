# Mocha & Chai

Mocha 提供测试运行器，Chai 提供断言语法。

# Mocha

在 **Mocha** 里，`describe` 和 `it` 是两个核心函数，用来组织和编写测试用例。

## `describe`

- **用途**：定义一个测试套件（Test Suite）。
- **作用**：把一组相关的测试组织在一起，方便管理和阅读。
- **特点**：
    - 通常对应一个功能模块或一个场景。
    - 可以嵌套，形成层级结构。
- **语法**：
    
    ```tsx
    describe("auction", () => {
      before(() => console.log("before"));          // 1次
      beforeEach(() => console.log("beforeEach"));  // 每个 it 之前 1次
      afterEach(() => console.log("afterEach"));    // 每个 it 之后 1次
      after(() => console.log("after"));            // 1次
    
      it("case1", () => console.log("it case1"));
      it("case2", () => console.log("it case2"));
    });
    ```
    

## `it`

- **用途**：定义一个具体的测试用例（Test Case）。
- **作用**：描述并验证某个具体功能是否按预期工作。
- **特点**：
    - 每个 `it` 都是一个独立的测试单元。
    - 建议用一句话描述预期行为。
- **语法**：
    
    ```tsx
    it("应该正确初始化平台", async () => {
      // 在这里写断言，比如 assert.ok(...)
    });
    ```
    

## `Before`

在运行一个 `describe` 里的所有 `it` 之前，先执行一次准备工作，如初始化等。

每次运行测试的时候，`before()` 会在所有 `it` 之前运行一次。

如果你在测试文件里有多个 `describe`，每个 `describe` 的 `before()` 都会独立跑一次。

## `After`

在运行一个 `describe` 里的所有 `it` 之后，执行一次。

## `beforeEach`

在 **每个 it 执行之前**，都会跑一次。

## `afterEach`

在 **每个 it 执行之后**，都会跑一次。

Mocha 默认是 **串行执行**，多个 `describe` 会按照它们在文件中的书写顺序依次执行，单个 `describe` 内部的 `it` 也是严格按声明顺序。

# Chai

Chai 是 JavaScript/TypeScript 世界里常用的一个 **断言库（Assertion Library）**，经常和 **Mocha** 搭配使用，用来编写单元测试或集成测试。它的目标是让测试代码更简洁、可读性更高。

Chai 提供了三种风格，你可以根据习惯选择：

- **Assert 风格**（类 C 风格，函数式断言）
- **Expect 风格**（语义化更强）
- **Should 风格**（链式断言，贴近自然语言）

---

[返回课程首页](../../../README.md)
