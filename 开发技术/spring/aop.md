版本：5.3.27

<br>

aop：面向切面编程

<br>

概念：

aspect（切面）：横跨多个类的关注点的模块化。通俗理解即将多个类中相同操作提取出来整合到一个单独的类。

join point（连接点）：程序运行中的一个点。在spring aop中其总是表示方法的执行。通俗理解即操作原本应该运行的位置。

advice（通知）：某个切面在某个连接点执行的操作。通知包括环绕、前置、后置。通俗理解即方法的增强操作。

pointcut（切点）：匹配连接点的的谓词表达式。通知将在与切点匹配的连接点上运行。

introduction（简介）：

target object（目标对象）：被通知的对象。

aop proxy（aop代理）：由aop框架创建的对象，用于实现aop。在sping中aop代理是jdk动态代理或 cglib代理。

weaving（织入）：将切面与对象连接并创建一个通知（代理）对象。它可以在编译、加载或运行时执行。在spring中是在运行时执行的。

<br>

实现类：AbstractAdvisorAutoProxyCreator

这是一个后置处理器，bean对象将在初始化后被代理（调用postProcessAfterInitialization）。
