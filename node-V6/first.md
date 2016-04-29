#example of Nodejs v6.0 for ES6
copy and edit from http://node.green/ for easy read and learn.Moreover, I will delete some trivial feature by my own mind. Everyone's PR is welcome!

##function
###class
class statement:
```javascript
  function () {
    class C {}
    return typeof C === "function"
  }
``` 
is block-scoped:
```javascript
  function () {
    class C {}
    let c1 = C
    {
      class C {}
      let c2 = C
    }
    return C === c1
  }
```
class expression:
anonymous class :
```javascript
  function () {
    return typeof class C {} === 'function'
  }
```
constructor:
```javascript
  function () {
    class C {
      constructor () { this.x = 1}
    }
    return C.prototype.constructor === C && new C().x === 1
  }
  return 
```
prototype methods:
```javascript
  function () {
    class C {
      method () { return 2 }
    }
    return typeof C.prototype.method === 'function' && new C().method() === 2
  }
```
string-keyed methods:
```javascript
  function () {
    class C {
      'foo bar' () { return 2 }
    }
    return typeof C.prototype['foo bar'] === 'function' && new C()['foo bar']() === 2
  }
```
computed prototype methods:
```javascript
  function () {
    let foo = 'method'
    class C {
      [foo] () { return 2 }
    }
    return typeof C.prototype.method === 'function' && new C().method() === 2
  }
```
static methods:
```javascript
  function () {
    class C { 
      static method () { return 3 }
    }
    return typeof C.method === 'function' && C.method() === 3 
  }
```
string-keyed static methods:
computed static methods:
some as prototype methods, just need a static keyword

accessor properties:
```javascript
  function () {
    let baz = false
    class C {
      get foo () { return 'foo' }
      set bar (x) { baz = x }
    }
    new C().bar(true)
    return new C().foo === 'foo' && baz
  }
```
string-keyed accessor properties:
computed accessor properties:
some as accessor properties, just need a get/set keyword

static accessor properties:
string-keyed static accessor properties:
computed static accessor properties:
some as accessor properties, just need a static and get/set keyword

class name is lexically scoped
```javascript
  function () {
    class C {
      method () { return typeof C === 'function'}
    }
    let M = C.prototype.method
    C = undefined
    return C === undefined && M()
  }
```
extends
```javascript
  function () {
    class B () {}
    class C extends B {
    }
    return new C() instanceof B && B.isPrototypeOf(C)    
  }
```
extends expressions
```javascript
  function () {
    let B
    class C extends (B = class {})
    return new C() instanceof B && B.isPrototypeOf(C)    
  }
```
extends null
```javascript
  function () {
    class C extends null {
      constructor () { return Object.create(null) }
    }
    return function.prototype.isPrototypeOf(C) && Object.getPrototypeOf(C.prototype) === null
  }
```
new.target
```javascript
  function () {
    let passed = false
    new function f () {
      passed = new.target === f
    }()
    class A {
      constructor () { passed &= new.target === B}
    }
    class B extends A {}
    new B()
    return passed
  }
```
##super
statement in constructors:
```javascript
  function () {
    let passed =  false
    class B {
      constructor (a) { passed = (a === 'barbaz')}
    }
    class C extends B {
      constructor (a) { super("bar" + a) }
    }
    new C("baz")
    return passed
  }
```
```
note: if you extends from B, you must call super in constructor of C
expression in constructor
```
```javascript
  function () {
    function () {
      class B {
        constructor (a) { return ['foo' + a]}
      }
      class C extends B {
        constructor (a) { return super('bar' + a)}
      }
      return new C('baz')[0] === 'foobarbaz'
    }
  }
```

```javascript
```
```javascript
```
```javascript
```
```javascript
```
```javascript
```
```javascript
```
