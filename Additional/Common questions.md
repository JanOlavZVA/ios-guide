
# @synthesize vs @dynamic
@synthesize will generate getter and setter methods for your property. @dynamic just tells the compiler that the getter and setter methods are implemented not by the class itself but somewhere else (like the superclass or will be provided at runtime).

Uses for @dynamic are e.g. with subclasses of NSManagedObject (CoreData) or when you want to create an outlet for a property defined by a superclass that was not defined as an outlet.

@dynamic also can be used to delegate the responsibility of implementing the accessors. If you implement the accessors yourself within the class then you normally do not use @dynamic.

Super class:
```objc
@property (nonatomic, retain) NSButton *someButton;
...
@synthesize someButton;
```

Subclass:
```objc
@property (nonatomic, retain) IBOutlet NSButton *someButton;
...
@dynamic someButton;
```

Dynamic is the default if you don't set either @synthesize or @dynamic. 
@dynamic means to responsibility of implementing the accessors is delegated. If you implement the accessors yourself within the class then you normally do not use @dynamic.

# Multiple Delegate
Objective-C работает с сообщениями. Мы не вызываем метод на объекте. Вместо этого, мы шлем ему сообщение. Таким образом, под Message Forwarding понимается редирект сообщения другому объекту, т.е. его проксирование.

Важно отметить, что отправка объекту сообщения, на которое он не отвечает, дает ошибку. Однако перед тем, как ошибка будет сгенерирована, рантайм даст объекту еще один шанс, чтобы обработать сообщение. 

1. Если объект реализует метод, т.е можно получить IMP (например, при помощи method_getImplementation(class_getInstanceMethod(subclass, aSelecor))), то рантайм вызывает метод. В противном случае, идем дальше.

2. Вызывается +(BOOL)resolveInstanceMethod:(SEL)aSEL или +(BOOL)resolveClassMethod:(SEL)name, если шлем сообщение классу. Этот метод дает возможность **добавить нужный селектор динамически.** Если возвращается YES, то рантайм сново пытается получить IMP и вызвать метод. В противном случае, идем дальше.

Еще данный метод вызывается при +(BOOL)respondsToSelector:(SEL)aSelector и +(BOOL)instancesRespondToSelector:(SEL)aSelector, если селектор не реализован. Причем, данный метод **вызывается только один раз для каждого селектора,** второго шанса добавить метод не будет!

```objc
+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
    if (aSEL == @selector(resolveThisMethodDynamically))
    {
          class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
          return YES;
    }
    return [super resolveInstanceMethod:aSel];
}
```

3. Выполняется так называемый Fast Forwarding. А именно, вызывается метод -(id)forwardingTargetForSelector:(SEL)aSelector
Этот метод возвращает объект, который надо использовать вместо текущего. В общем-то, очень удобная штука для имитации **множественного наследования**. Fast он, потому что на данном этапе можно сделать форвардинг без создания NSInvoacation.

Для возвращенного этим методом объекта будут повторены все шаги. Согласно документации, если вернуть self, то будет бесконечный цикл. На практике, бесконечного цикла не возникает: видимо, в рантайм внесли поправки.

4. Два предыдущих шага являются оптимизацией форвардинга. После них рантайм создает NSInvocation.
```objc
NSMethodSignature *sig = ...
NSInvocation* inv = [NSInvocation invocationWithMethodSignature:sig];
[inv setSelector:selector];
```
Т.е для создания NSInvocation, рантайму надо получить сигнатуру метода (NSMethodSignature). Поэтому у объекта вызывается ```- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector```. Если метод вместо NSMethodSignature вернет nil, то рантайм вызовет у объекта ```-(void)doesNotRecognizeSelector:(SEL)```aSelector, т.е. произойдет крэш.

Создать NSMethodSignature можно следующими способами:
* использовать метод ```+(NSMethodSignature *)instanceMethodSignatureForSelector:(SEL)aSelector```
Заметьте, если класс всего лишь заявляет, что реализует протокол (@interface MyClass : NSObject ), то этот методу уже вернет не nil, а сигнатуру.
* использовать метод ```-(NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector```
Внутри вызывает ```[[self class] instanceMethodSignatureForSelector:...]```
* использовать метод ```+(NSMethodSignature *)signatureWithObjCTypes:(const char *)types``` принадлежащий классу ```NSMethodSignature``` и собрать ```NSMethodSignature``` самому

# What is meta-class in Objective-C?

In Objective-C, an object's class is determined by its isa pointer. The isa pointer points to the object's Class.

In fact, the basic definition of an object in Objective-C looks like this:
```objc
typedef struct objc_object {
    Class isa;
} *id;
```
What this says is: any structure which starts with a pointer to a Class structure can be treated as an objc_object.

The most important feature of objects in Objective-C is that you can send messages to them:
```[@"stringValue" writeToFile:@"/file.txt" atomically:YES encoding:NSUTF8StringEncoding error:NULL];```
This works because when you send a message to an Objective-C object (like the ```NSCFString``` here), the runtime follows object's ```isa``` pointer to get to the object's ```Class``` (the ```NSCFString``` class in this case). The ```Class``` then contains a list of the Methods which apply to all objects of that ```Class``` and a pointer to the superclass to look up inherited methods. The runtime looks through the list of Methods on the ```Class``` and superclasses to find one that matches the message selector (in the above case, ```writeToFile:atomically:encoding:error``` on ```NSString```). The runtime then invokes the function ```(IMP)``` for that method.

The important point is that the ```Class``` defines the messages that you can send to an object.

## What is a meta-class?

Now, as you probably already know, a Class in Objective-C is also an object. This means that you can send messages to a Class.

```NSStringEncoding defaultStringEncoding = [NSString defaultStringEncoding];```
In this case, defaultStringEncoding is sent to the NSString class.

This works because every Class in Objective-C is an object itself. This means that the Class structure must start with an isa pointer so that it is binary compatible with the objc_object structure I showed above and the next field in the structure must be a pointer to the superclass (or nil for base classes).

```Class``` starts with an ```isa``` field followed by a ```superclass``` field.
```objc
typedef struct objc_class *Class;
struct objc_class {
    Class isa;
    Class super_class;
    /* followed by runtime specific details... */
};
```
```objc
typedef struct objc_selector *SEL;
typedef struct objc_method *Method;
typedef struct objc_ivar *Ivar;
typedef struct objc_category *Category;
typedef struct objc_property *objc_property_t;
```
However, in order to let us invoke a method on a ```Class```, the ```isa``` pointer of the Class must itself point to a Class structure and that Class structure must contain the list of Methods that we can invoke on the Class.

This leads to the definition of a meta-class: the meta-class is the class for a Class object.

* When you send a message to an object, that message is looked up in the method list on the object's class.
* When you send a message to a class, that message is looked up in the method list on the class' meta-class.

The meta-class is essential because it stores the class methods for a Class. There must be a unique meta-class for every Class because every Class has a potentially unique list of class methods.

## What is the class of the meta-class?

The meta-class, like the Class before it, is also an object. This means that you can invoke methods on it too. Naturally, this means that it must also have a class.

All meta-classes use the base class' meta-class (the meta-class of the top Class in their inheritance hierarchy) as their class. This means that for all classes that descend from NSObject (most classes), the meta-class has the NSObject meta-class as its class.

Following the rule that all meta-classes use the base class' meta-class as their class, any base meta-classes will be its own class (their isa pointer points to themselves). This means that the isa pointer on the NSObject meta-class points to itself (it is an instance of itself).

## Inheritance for classes and meta-classes

In the same way that the Class points to the superclass with its super_class pointer, the meta-class points to the meta-class of the Class' super_class using its own super_class pointer.

As a further quirk, the base class' meta-class sets its super_class to the base class itself.

The result of this inheritance hierarchy is that all instances, classes and meta-classes in the hierarchy inherit from the hierarchy's base class.

For all instances, classes and meta-classes in the NSObject hierarchy, this means that all NSObject instance methods are valid. For the classes and meta-classes, all NSObject class methods are also valid.

<img src="https://github.com/m4stodon/ios-guide/tree/master/Additional/Images/Class_inheritance.png"/>

The meta-class is the class for a Class object. Every Class has its own unique meta-class (since every Class can have its own unique list of methods). This means that all Class objects are not themselves all of the same class.

The meta-class will always ensure that the Class object has all the instance and class methods of the base class in the hierarchy, plus all of the class methods in-between. For classes descended from NSObject, this means that all the NSObject instance and protocol methods are defined for all Class (and meta-class) objects.

All meta-classes themselves use the base class' meta-class (NSObject meta-class for NSObject hierarchy classes) as their class, including the base level meta-class which is the only self-defining class in the runtime.

https://www.cocoawithlove.com/2010/01/what-is-meta-class-in-objective-c.html

# Meta-classes in Swift
## Type.self
The meta type of a type can be found by calling .self on a type. The returned type can either be a Class meta-type, a Struct meta-type or a Protocol meta-type.
```swift
String.self // String.Type
NSString.self // NSString.Type
MyClass.self // MyClass.Type
```
Even if the above look similar, they act different. The type return from calling NSString.self looks similar to an Objective C class. However, it has other methods defined in swift extensions, for example, calling NSString.self.className() returns “NSString”
The object returned from calling MyClass.self is the swift metaType of MyClass. This object exposes init function and all the method defined in this class as curried methodю

## Swift Currying
>An instance method in Swift is just a type method that takes the instance as an argument and returns a function which will then be applied to the instance.

For example, suppose you have function of type ```(Int, Double) -> String``` — it takes an ```Int``` and a ```Double```, and returns a ```String```. If we curry this function, we get ```(Int) -> (Double) -> (String)```, i.e. a function that takes just an ```Int``` and returns a second function. This second function then takes the ```Double``` argument and returns the final ```String```. To call the curried function, you’d chain two function calls:

```swift
let result: String = f(42)(3.14) // f takes an Int and returns a function that takes a Double.
```

Why would you want to do this? The big advantage of curried functions is that they can be partially applied, i.e. some arguments can be specified (bound) before the function is ultimately called. Partial function application yields a new function that you can then e.g. pass around to another part of your code. Languages like Haskell and ML use currying for all functions.

Swift uses the same idea of partial application for instance methods. Although it’s not really accurate to say instance methods in Swift are “curried”, I still like to use this term.

**Example**
```swift
class BankAccount {
    var balance: Double = 0.0

    func deposit(_ amount: Double) {
        balance += amount
    }
}
```
```
let account = BankAccount()
account.deposit(100) // balance is now 100

//So far, so simple. But we can also do this:

let depositor = BankAccount.deposit(_:)
depositor(account)(100) // balance is now 200

//This is totally equivalent to the above. 
```
What’s going on here? We first assign the method to a variable. Note that we didn’t pass an argument after ```BankAccount.deposit(_:)``` — we’re not calling the method here (which would yield an error because you can’t call an instance method on the type), just referencing it, much like a function pointer in C. The second step is then to call the function stored in the depositor variable. Its type is as follows:

```swift
let depositor: BankAccount -> (Double) -> ()
```
In other words, this function has a single argument, a BankAccount instance, and returns another function. This latter function takes a Double and returns nothing. You should recognize the signature of the ```deposit()``` instance method in this second part.

I hope you can see that an instance method in Swift is simply a type method that takes the instance as an argument and returns a function which will then be applied to the instance. Of course, we can also do this in one line, which makes the relationship between type methods and instance methods even clearer:

```swift
BankAccount.deposit(account)(100) // balance is now 300
```
By passing the instance to ```BankAccount.deposit()```, the instance gets bound to the function. In a second step, that function is then called with the other arguments. Pretty cool, eh?

## Back to meta

If we pass MyClass.self to NSStringFromClass we get the mangled class name.
```
NSStringFromClass(MyClass.self) // __lldb_expr_368__.MyClass
```
String.self returns a similar to calling self on a swift class. A difference to note here, if we pass it to NSStringFromClass we get a compilation error. This should not be surprising as string is a struct and not a class.

## dynamicType
If .self is used on a Type to returns its metatype, dynamicType is used on an instance to return its type. The type returned is the same returned when calling .self on the instance static type.
Comparing the dynamic type to the static type for most of the struct and types; we can compare instances of MyClass as follows:
```
MyClass.self = MyClass().dynamicType // true
```

We can create a new instance from a type. For example to create an instance from MyClass we can use the following:
```
MyClass.self.init()
```

The above won’t work for the dynamicType.
As the error describes, the solution is to add a required init to MyClass:
```
MyClass().dynamicType.init() // error

class MyClass() {
	required init() {} // solves the problem
}
```

* Variable annotated with String.Type or any Struct.Type can only store that struct meta-type. The compiler won’t allow us to store another metatype to the same variable
* For classes, since they can be subclassed, the behaviour is different. A BaseClass.Type can also hold references of SubClass.Type.
* A Protocol.Type variable, unsurprisingly, can hold a reference to any type thats implementing that protocol


# Objective-C Runtime
Помимо определения основных структур языка, библиотека включает в себя набор функций, работающих с этими структурами. Их можно условно разделить на несколько групп (назначение функций, как правило, очевидно из их названия):
Манипулирование классами: class_addMethod, class_addIvar, class_replaceMethod
Создание новых классов: class_allocateClassPair, class_registerClassPair
Интроспекция: class_getName, class_getSuperclass, class_getInstanceVariable, class_getProperty, class_copyMethodList, class_copyIvarList, class_copyPropertyList
Манипулирование объектами: objc_msgSend, objc_getClass, object_copy
Работа с ассоциативными ссылками

## Пример 1. Интроспекция объекта

```objc
@interface COConcreteObject : COBaseObject
@property(nonatomic, strong) NSString *name;
@property(nonatomic, strong) NSString *title;
@property(nonatomic, strong) NSNumber *quantity;
@end
```

Для удобства отладки хотелось бы, чтобы при выводе в лог печаталась информация о состоянии свойств объекта, а не нечто вроде <COConcreteObject: 0x71d6860>. Поскольку модель данных достаточно разветвленная, с большим количеством различных подклассов, нежелательно писать для каждого класса отдельный метод description, в котором вручную собирать значения его свойств. На помощь приходит Objective-C Runtime:
```objc
@implementation COBaseObject

- (NSString *)description {
    NSMutableDictionary *propertyValues = [NSMutableDictionary dictionary];
    unsigned int propertyCount;
    objc_property_t *properties = class_copyPropertyList([self class], &propertyCount);
    for (unsigned int i = 0; i < propertyCount; i++) {
        char const *propertyName = property_getName(properties[i]);
        const char *attr = property_getAttributes(properties[i]);
        if (attr[1] == '@') {
            NSString *selector = [NSString stringWithCString:propertyName encoding:NSUTF8StringEncoding];
            SEL sel = sel_registerName([selector UTF8String]);
            NSObject * propertyValue = objc_msgSend(self, sel);
            propertyValues[selector] = propertyValue.description;
        }
    }
    free(properties);
    return [NSString stringWithFormat:@"%@: %@", self.class, propertyValues];
}

@end
```
Метод, определенный в общем суперклассе объектов модели, получает список всех свойств объекта с помощью функции class_copyPropertyList. Затем значения свойств собираются в NSDictionary, который и используется при построении строкового представления объекта. Данный алгоритм раработает только со свойствами, которые являются Objective-C объектами. Проверка типа осуществляется с использованием функции property_getAttributes. Результат работы метода выглядит примерно так:

2013-05-04 15:54:01.992 Test[40675:11303] COConcreteObject: {
name = Foo;
quantity = 10;
title = bar;
}

## Сообщения

Система вызова методов в Objective-C реализована через посылку сообщений объекту. Каждый вызов метода транслируется в соответствующий вызов функции objc_msgSend:
```objc
// Вызов метода
[array insertObject:foo atIndex:1];
// Соответствующий ему вызов Runtime-функции
objc_msgSend(array, @selector(insertObject:atIndex:), foo, 1);
```
Вызов objc_msgSent инициирует процесс поиска реализации метода, соответствующего селектору, переданному в функцию. Реализация метода ищется в так называемой таблице диспетчеризации класса. Поскольку этот процесс может быть достаточно продолжительным, с каждым классом ассоциирован кеш методов. После первого вызова любого метода, результат поиска его реализации будет закеширован в классе. Если реализация метода не найдена в самом классе, дальше поиск продолжается вверх по иерархии наследования — в суперклассах данного класса. Если же и при поиске по иерархии результат не достигнут, в дело вступает механизм динамического поиска — вызывается один из специальных методов: resolveInstanceMethod или resolveClassMethod. Переопределение этих методов — одна из последних возможностей повлиять на Runtime:
```objc
+ (BOOL)resolveInstanceMethod:(SEL)aSelector {
    if (aSelector == @selector(myDynamicMethod)) {
        class_addMethod(self, aSelector, (IMP)myDynamicIMP, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:aSelector];
}
```
Здесь вы можете **динамически указать свою реализацию вызываемого метода.** Если же этот механизм по каким-то причнам вас не устраивает — вы можете использовать форвардинг сообщений.

## Пример 2. Method Swizzling

Одна из особенностей категорий в Objective-C — метод, определенный в категории, полностью перекрывает метод базового класса. Иногда нам требуется не переопределить, а расширить функционал имеющегося метода. Пусть, например, по каким-то причинам нам хочется залогировать все добавления элементов в массив NSMutableArray. Стандартными средствами языка этого сделать не получится. Но мы можем использовать прием под названием method swizzling:
```objc
@implementation NSMutableArray (CO)

+ (void)load {
    Method addObject = class_getInstanceMethod(self, @selector(addObject:));
    Method logAddObject = class_getInstanceMethod(self, @selector(logAddObject:));
    method_exchangeImplementations(addObject, logAddObject);
}

- (void)logAddObject:(id)aObject {
    [self logAddObject:aObject];
    NSLog(@"Добавлен объект %@ в массив %@", aObject, self);
}

@end
```
Мы перегружаем метод ```load``` — это специальный callback, который, если он определен в классе, будет вызван во время инициализации этого класса — до вызова любого из других его методов. Здесь мы меняем местами реализацию базового метода addObject: и нашего метода logAddObject:. Обратите внимание на «рекурсивный» вызов в logAddObject: — это и есть обращение к перегруженной реализации основного метода. 

## Пример 3. Ассоциативные ссылки

Еще одним известным ограничением категорий является невозможность создания в них новых переменных экземпляра. Пусть, например, вам требуется добавить новое свойство к библиотечному классу UITableView — ссылку на «заглушку», которая будет показываться, когда таблица пуста:
```objc
@interface UITableView (Additions)

@property(nonatomic, strong) UIView *placeholderView;

@end
```
«Из коробки» этот код работать не будет, вы получите исключение во время выполнения программы. Эту проблему можно обойти, используя функционал ассоциативных ссылок:
```objc
static char key;

@implementation UITableView (Additions)

-(void)setPlaceholderView:(UIView *)placeholderView {
    objc_setAssociatedObject(self, &key, placeholderView, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

-(UIView *) placeholderView {
    return objc_getAssociatedObject(self, &key);
}

@end
```
Любой объект вы можете использовать как ассоциативный массив, связывая с ним другие объекты с помощью функции objc_setAssociatedObject. Для ее работы требуется ключ, по которому вы потом сможете извлечь нужный вам объект назад, используя вызов objc_getAssociatedObject. При этом вы не можете использовать скопированное значение ключа — это должен быть именно тот объект (в примере — указатель), который был передан в вызове objc_setAssociatedObject. 


# NSObject ```+load``` and ```+initialize```
## The ```load``` message
The runtime sends the load message to each class object, very soon after the class object is loaded in the process's address space. For classes that are part of the program's executable file, the runtime sends the load message very early in the process's lifetime. For classes that are in a shared (dynamically-loaded) library, the runtime sends the load message just after the shared library is loaded into the process's address space.

The load message is sent to classes and categories that are both dynamically loaded and statically linked, but only if the newly loaded class or category implements a method that can respond.
The order of initialization is as follows:
* All initializers in any framework you link to.
* All +load methods in your image.
* All C++ static initializers and C/C++ __attribute__(constructor) functions in your image.
* All initializers in frameworks that link to you.
In addition:
- A class’s +load method is called after all of its superclasses’ +load methods.
- A category +load method is called after the class’s own +load method.
In a custom implementation of load you can therefore safely message other unrelated classes from the same image, but any load methods implemented by those classes may not have run yet.

Furthermore, the runtime only sends load to a class object if that class object itself implements the load method. Example:
```objc
@interface Superclass : NSObject
@end

@interface Subclass : Superclass
@end

@implementation Superclass
+ (void)load {
    NSLog(@"in Superclass load");
}
@end

@implementation Subclass
// ... load not implemented in this class
@end
```
* The runtime sends the load message to the Superclass class object. It does not send the load message to the Subclass class object, even though Subclass inherits the method from Superclass.
* The runtime sends the load message to a class object after it has sent the load message to all of the class's superclass objects (if those superclass objects implement load) and all of the class objects in shared libraries you link to. But you don't know which other classes in your own executable have received load yet.
* Every class that your process loads into its address space will receive a load message, if it implements the load method, regardless of whether your process makes any other use of the class.
* You can see how the runtime looks up the load method as a special case in the ```_class_getLoadMethod``` of objc-runtime-new.mm, and calls it directly from ```call_class_loads``` in objc-loadmethod.mm
* **The runtime also runs the load method of every category it loads, even if several categories on the same class implement load.** This probably goes against everything you know about how categories work, but that's because +load is not a normal method. This feature means that +load is an excellent place to do evil things like method swizzling.

**Objective-C runtime handles +load and +initialize methods differently.**
* +load will be called only once on each class and each class category only if class (or category) explicitly implements this method.
* +initialize is called in more traditional manner – if category implements +initialize, it will override implementation of the class. Also, if subclass does not implement +initialize runtime will call parent’s implementation, if any. This might cause +initialize to be called several times, and developer should protect his code from this.

These differences mean that you should not make any “protections” in +load method, but you MUST check class in +initialize:
```objc
@interface MyClass : NSObject
@end
 
@implementation MyClass
+(void)initialize
{
    if (self == [MyClass class]) 
    {
        // do initialization
    }
}
@end
```
You should also avoid any time consuming operations in +load (and in +initialize too). Since this +load is called in application start sequence – you need to save as much time as possible, or your app could be killed.

### Swizzling
One example of +load usage is to swizzle methods.

Objective-C runtime offers tons of APIs for developers to manage classes. One of those is – to replace one method implementation with another. I won’t go into details, let’s just list some of data types:
* Class – defines class object, used to access class properties ([object class] will return Class object).
* SEL – selector, method signature like setTitle:forState:, different classes could have methods with identical selectors.
* IMP – method implementation, points to the actual code which is being executed.
* Method – method object, includes all information about method, including selector and implementation and class it belongs to.
And just two functions:
* class_getInstanceMethod – returns Method for specific Class and SEL.
* method_exchangeImpelementations – exchange IMP‘s of two Method‘s passed to this function.

Let’s look at example. We’re defining a category on UIButton to log all title changes in your application.
```objc
@implementation UIButton (Logger)
 
-(void)logAndSetTitle:(NSString*)title forState:(UIControlState)state
{
    NSLog(@"changing title [%@] to [%@] (state %d)", self.titleLabel.text, title, state);
    [self logAndSetTitle:title forState:state];
}
 
+(void)load
{
    Method setTitleOrig = class_getInstanceMethod(self, @selector(setTitle:forState:));
    Method setTitleLog = class_getInstanceMethod(self, @selector(logAndSetTitle:forState:));
    method_exchangeImplementations(setTitleOrig, setTitleLog);
}
```
We’re defining new method ```-logAndSetTitle:forState:``` which has the same parameters as UIButton‘s ```-setTitle:forState:```.

In category’s +load swizzling these methods. This means that when any code is calling -setTitle:forState: on any UIButton, implementation of -logAndSetTitle:forState: will be called.

## The ```initialize``` Method

The runtime calls the initialize method on a class object just before sending the first message (other than load or initialize) to the class object or any instances of the class. This message is sent using the normal mechanism, so if your class doesn't implement initialize, but inherits from a class that does, then your class will use its superclass's initialize. The runtime will send the initialize to all of a class's superclasses first (if the superclasses haven't already been sent  initialize).

```+initialize``` is interesting because it's invoked lazily and may not be invoked at all. When a class first loads, +initialize is not called. When a message is sent to a class, the runtime first checks to see if +initialize has been called yet. If not, it calls it before proceeding with the message send.

Example:
```objc
@interface Superclass : NSObject
@end

@interface Subclass : Superclass
@end

@implementation Superclass

+ (void)initialize {
    NSLog(@"in Superclass initialize; self = %@", self);
}

@end

@implementation Subclass

// ... initialize not implemented in this class

@end

int main(int argc, char *argv[]) {
    @autoreleasepool {
        Subclass *object = [[Subclass alloc] init];
    }
    return 0;
}
```
This program prints two lines of output:
```
2012-11-10 16:18:38.984 testApp[7498:c07] in Superclass initialize; self = Superclass
2012-11-10 16:18:38.987 testApp[7498:c07] in Superclass initialize; self = Subclass
```
Since the system sends the initialize method lazily, a class won't receive the message unless your program actually sends messages to the class (or a subclass, or instances of the class or subclasses). And by the time you receive initialize, every class in your process should have already received load (if appropriate).

The canonical way to implement initialize is this:
```objc
@implementation Someclass

+ (void)initialize {
    if (self == [Someclass class]) {
        // do whatever
    }
}
```
**The point of this pattern is to avoid Someclass re-initializing itself when it has a subclass that doesn't implement initialize.**

The runtime sends the initialize message in the ```_class_initialize``` function in objc-initialize.mm. You can see that it uses ```objc_msgSend``` to send it, which is the normal message-sending function.



# How to run your thread with runloop
```objc
-(void)viewDidLoad {
    [super viewDidLoad];
    NSThread* myThread = [[NSThread alloc] initWithTarget:self
                                                 selector:@selector(threadMain)
                                                   object:nil];
    [myThread start];  // Actually create the thread
}

-(void)threadMain {
    @autoreleasepool {
        NSRunLoop *TimerRunLoop = [NSRunLoop currentRunLoop];
        timer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(timerMethod) userInfo:nil repeats:YES];
        [TimerRunLoop run];
    }
}

-(void)timerMethod {
    NSLog(@"Timer fired");
    //[timer invalidate];
}
```

# What is the point of categories?
The main reason for categories is to allow you to add methods to a class for which you don't have the source code, or for which you don't want to modify the source code.

Example 1.

I wanted a method to create an animated UIImage by loading an animated GIF. Logically this should be a UIImage class method, but I don't have the source code for the UIKit framework (which contains UIImage). So I wrote a category for UIImage that adds a class method named animatedImageWithAnimatedGIFData:. You can find it in my uiimage-from-animated-gif repository.

Did I have to add this method to UIImage? No. I could have made it a regular C function, or I could have made a utility class (perhaps named AnimatedGIFLoader) to hold the method. But from a design standpoint, the method logically belongs on UIImage.

Example 2.

Apple wanted to make it easy to draw a string into a graphics context. In a program with a GUI, it would be reasonable for NSString to have a draw method. Apple has the source code to the Foundation framework (which contains NSString), so they could add it. But the Foundation framework is designed to be used in all sorts of programs, including programs that don't have any user interface. So the classes in Foundation don't know anything about UIKit or AppKit or Core Graphics or any other higher-level library that can draw graphics.

Instead, UIKit has a category that adds the drawAtPoint:withFont: method to NSString. AppKit has a category that adds the drawAtPoint:withAttributes: method to NSString.

AppKit and UIKit have a number of other categories that add methods to Foundation classes. For example, UIKit has categories on NSObject, NSIndexPath, NSCoder, and more.

Another reason to use a category is to **split up the implementation of a class into multiple .m files.** If you have a big class, you can move some of its selectors into a category and implement the category methods in a separate source file. **The linker will automatically merge the category into the class when it creates the executable file, so there is no run-time penalty.**


# NSPointerArray
NSPointerArray is an alternative to Array with the main difference that it doesn’t store an object but its pointer (UnsafeMutableRawPointer).

This type of array can store the pointer maintaining either a weak or a strong reference depending on how it’s initialized. It provides two static methods to be initialized in different ways:

```objc
let strongRefarray = NSPointerArray.strongObjects() // Maintains strong references
let weakRefarray = NSPointerArray.weakObjects() // Maintains weak references
 ```
Simplify using of NSPointerArray:
```objc
extension NSPointerArray {
    func addObject(_ object: AnyObject?) {
        guard let strongObject = object else { return }
 
        let pointer = Unmanaged.passUnretained(strongObject).toOpaque()
        addPointer(pointer)
    }
 
    func insertObject(_ object: AnyObject?, at index: Int) {
        guard index < count, let strongObject = object else { return }
 
        let pointer = Unmanaged.passUnretained(strongObject).toOpaque()
        insertPointer(pointer, at: index)
    }
 
    func replaceObject(at index: Int, withObject object: AnyObject?) {
        guard index < count, let strongObject = object else { return }
 
        let pointer = Unmanaged.passUnretained(strongObject).toOpaque()
        replacePointer(at: index, withPointer: pointer)
    }
 
    func object(at index: Int) -> AnyObject? {
        guard index < count, let pointer = self.pointer(at: index) else { return nil }
        return Unmanaged<AnyObject>.fromOpaque(pointer).takeUnretainedValue()
    }
 
    func removeObject(at index: Int) {
        guard index < count else { return }
 
        removePointer(at: index)
    }
}
```

**Alternative**
A possible workaround is creating a new class WeakRef with a generic weak property value:
```objc
class WeakRef<T> where T: AnyObject {
 
    private(set) weak var value: T?
 
    init(value: T?) {
        self.value = value
    }
}
```

If you need something similar to ```NSPointerArray``` for ```Dictionary``` you can have a look at ```NSMapTable```, whereas for ```Set``` you can use ```NSHashTable```.

If you want a type-safe ```Dictionary```/```Set```, you can achieve it storing a ```WeakRef``` object.


# Swift reference types

## STRONG
Let's start off with what a strong reference is. It's essentially a normal reference (pointer and all), but it's special in it's own right in that it protects the referred object from getting deallocated by ARC by increasing it's retain count by 1. In essence, as long as anything has a strong reference to an object, it will not be deallocated.
```objc
class Kraken {
    let tentacle = Tentacle() //strong reference to child.
}
class Tentacle {
    let sucker = Sucker() //strong reference to child
}
```
Similarly, in animation blocks, the reference hierarchy is similar as well:
```objc
UIView.animate(withDuration: 0.3) {
    self.view.alpha = 0.0 // <- STRONG SELF
}
```

## WEAK
A weak reference is just a pointer to an object that doesn't protect the object from being deallocated by ARC. While strong references increase the retain count of an object by 1, weak references do not. In addition, weak references zero out the pointer to your object when it successfully deallocates. This ensures that when you access a weak reference, it will either be a valid object, or nil.
```objc
class Kraken {
    //let is a constant! All weak variables MUST be mutable.
    weak let tentacle = Tentacle() 
}
```

## UNOWNED
Weak and unowned references behave similarly but are NOT the same. Unowned references, like weak references, do not increase the retain count of the object being referred. However, in Swift, an unowned reference has the added benefit of not being an Optional. This makes them easier to manage rather than resorting to using optional binding. This is not unlike Implicitly Unwrapped Optionals. In addition, unowned references are non-zeroing. This means that when the object is deallocated, it does not zero out the pointer. This means that use of unowned references can, in some cases, lead to dangling pointers. **Swifts unowned references maps to Objective-C unsafe_unretained references.**

A really good place to use unowned references is when using self in closure properties that are lazily defined like so:
```objc
class Kraken {
    let petName = "Krakey-poo"
    lazy var businessCardName: (Void) -> String = { [unowned self] in
        return "Mr. Kraken AKA " + self.petName
    }
}
```

## Retain cycle
Important places to use weak variables are in cases where you have potential retain cycles. A retain cycle is what happens when two objects both have strong references to each other. If 2 objects have strong references to each other, ARC will not generate the appropriate release message code on each instance since they are keeping each other alive. Here's a neat little image from Apple that nicely illustrates this:

<img src="https://github.com/m4stodon/ios-guide/tree/master/Additional/Images/retain-cycle.png"/>

```objc
class Kraken {
    var notificationObserver: ((Notification) -> Void)?
    init() {
        notificationObserver = NotificationCenter.default.addObserver(forName: "humanEnteredKrakensLair", object: nil, queue: .main) { notification in
            self.eatHuman()
        }
    }

    deinit {            
        NotificationCenter.default.removeObserver(notificationObserver)
    }
}
```
Here, NotificationCenter retains a closure that captures self strongly when you call eatHuman(). Best practice says that you clear out notification observers in the deinit function. The problem here is that we don’t clear out the block until deinit, but deinit won’t ever be called by ARC because the closure has a strong reference to the Kraken instance!

Other gotchas where this could happen is in places like NSTimers and NSThread.

The fix is to use a weak reference to self in the closure's capture list.

<img src="https://github.com/m4stodon/ios-guide/tree/master/Additional/Images/retain-cycle-broken.png"/>

```objc
NotificationCenter.default.addObserver(forName: "humanEnteredKrakensLair", object: nil, queue: .main) { [weak self] notification in //The retain cycle is fixed by using capture lists!
    self?.eatHuman() //self is now an optional!
}
```

Also we can use weaks for properties:
```objc
class Tentacle {
    weak var delegate: LossOfLimbDelegate? // <--- keeps weak reference to delegate

    func cutOffTentacle() {
        delegate?.limbHasBeenLost()
    }
}
```

**Use a weak reference whenever it is valid for that reference to become nil at some point during its lifetime. Conversely, use an unowned reference when you know that the reference will never be nil once it has been set during initialization.**









