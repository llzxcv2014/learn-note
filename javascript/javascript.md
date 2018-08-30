# Javascript Note

## `this` 关键字

### 为何要使用 `this` 关键字

```java
function identify() {
    return this.name.toUpperCase();
}

function speak() {
    var greeting = "Hello, I'm " + identify.call( this );
    console.log( greeting );
}

var me = {
    name: "Kyle"
};

var you = {
    name: "Reader"
};
identify.call( me ); // KYLE
identify.call( you ); // READER
speak.call( me ); // Hello, I'm KYLE
speak.call( you ); // Hello, I'm READER
```

在不同的上下文对象中重复使用函数，不用针对每个对象编写不同的实现。
`this` 以一种更优雅的方式隐式“传递”一个对象的引用，可以将API设计得更加简洁。
```java
function foo(num) {
    console.log( "foo: " + num );
    // 记录 foo 被调用的次数
    this.count++;
} 
foo.count = 0;
var i;
for (i=0; i<10; i++) {
    if (i > 5) {
        foo(i);
    }
} 
// foo: 6
// foo: 7
// foo: 8
// foo: 9
// foo 被调用了多少次？
console.log( foo.count ); // 0
```

