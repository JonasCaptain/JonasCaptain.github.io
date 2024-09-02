---
layout: "@layouts/ArticleLayout.astro"
title: Python_Unittest
description: Python_Unittest 解读
date: 21 Feb 2023
tags:
  - Python
  - UnitTest
---

## 简介

> 从python官网中关于Unittest介绍中摘抄

**test fixture**: 表示执行一个或多个测试所需的准备，以及任何关联的清理操作。这可能涉及，例如，创建临时或代理数据库、目录或启动服务器进程。(这只是一个概念，在Unittest代码中并不存在具体的实现函数或者实现类，而是以一系列代码的组合来完成的)

**test case**: 最小的测试单元。它检查对特定输入集的特定响应。unittest提供了一个基类TestCase，它可以用来创建新的测试用例。

**test suite**：test suite 是test case、test suite或两者的集合。它用于聚合应该一起执行的测试。

**test runner**：runner是组织测试执行并向用户提供结果的组件。运行程序可以使用图形界面、文本界面或返回一个特殊值来指示执行测试的结果。

--------

## 目录结构

- `__init__.py`：通过 `__all__` 控制了导包的时候能导入哪些类
- `__mian__.py`：这个与Unittest框架关系不大，可以作为了解，有了此文件，可以将unittest目录当做一个Package使用，python -m unittest。详情可以查询一下，`__main__.py`文件的作用
- case.py：通过继承的方式，将子类中的test_打头的方法组合上setUp和tearDown形成 testFixture，以testFixture方式执行测试
- loader.py：负责加载各种标准方式的测试类，并将它们封装成TestSuite
- main.py：单元测试的执行入口
- result.py：管理测试过程中输出的信息(Error，Exception，Success，Skip)
- runner.py：测试执行器
- singals.py：处理执行过程中遇到的中断信号量
- suite.py：由Case组合起来的一个整体，可以批量执行Case
- util.py：工具函数，一些处理细节

--------

## 具体文件分析

> 仅分析文件中重要的函数、类以及其中的方法

### case.py

异常：

```python
class Skip(Exception):
    '''
     抛出异常被捕获，用于判断是否跳过该用例
     使用方法：
      通过装饰器，配置需要跳过的用例来达到目的
     装饰器函数(全局)：
      skip(reason)，表示无条件通过，reason用于说明跳过原因
      skipIf(condition, reason)，表示条件为真即跳过
      skipUnless(condition, reason)，表示条件不为真即跳过
    '''
class _ExpectedFailure(Exception):
    '''
     抛出异常被捕获，用于判断用例中是否出现了期望的<异常>
     使用方法:
      通过装饰器配置
     装饰器函数(全局):
      expectedFailure(func)
    '''
class _UnexpectedSuccess(Exception):
    '''
     抛出异常被捕获，用于判断结果期望失败，实际成功
     使用方法：
      与_ExpectedFailure一样
    '''

def expectedFailure(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        try:
            func(*args, **kwargs)
        except Exception:
            raise _ExpectedFailure(sys.exc_info())
        raise _UnexpectedSuccess
    return wrapper
    
```

类：

```python
class _AssertRaiseContext(object):
    '''
     断言上下文管理器(源码中使用频率很低)
    '''
class TestCase(object):
    def __init__(self, methodName='runTest'):
        '''
         methodName：就是需要执行的单元测试函数名，即def test_xxx(self)中的test_xxx
         如果TestCase类中没有methodName属性就会报错，通过继承的方式一般不会出现这个问题，除非是额外     的扩展unittest
         self._cleanups = []：这个属性是用来清理环境的，与teardownhi还不一样
         集成了常用的assert方法，目的是方便输出错误信息
        '''
    def addCleanup(self, function, *arg, **kwargs):
        '''
         添加一个function到self._cleanups
         添加到cleanups中的函数，会在teardown后执行，即使setup失败也会执行
         cleanups中的函数，采取的是后进先出的栈模型，实现原理就是对列表进行倒序排序
        '''
    def setUp(self):
        '''
         在每个测试方法之前执行
        '''
    def teardown(self):
        '''
         在每个测试方法之后执行
        '''
    def setUpClass(cls):
        '''
         在每个测试类执行前执行
        '''
    def tearDownClass(cls):
        '''
         在每个测试类执行之后执行
        '''
    def run(self, result=None):
        '''
         result：unittest中默认的一个类，用于收集测试过程中记录的结果，成功、失败、异常报错等信息
         result如果为None，则会使用unittest.result.TestResult()。TestResult中有一个空方法
         startTestRun()，如果方法被重写(继承)，将会被执行；默认情况下，这个方法就会在所有的测试执
         行前执行
         self._resultForDoCleanups：result的后续处理
         result.startTest(self)，执行统计testcase等一系列result的前置操作，这个方法在
         startTestRun执行后执行
         逻辑处理：
          1、判断class有没有被标记@unittest.skip
          2、判断method有没有被标记@unittest.skip
          3、确定需要跳过时，将结果记录在result中，退出测试
         Case执行流程：
          1、执行setUp()并处理异常：
           1、处理跳过逻辑
           2、处理键盘Ctrl+C退出
           3、处理其他Exception
          2、执行以test_打头的方法并处理异常：
           1、处理Ctrl+C
           2、处理断言异常
           3、处理预期抛出的异常
           4、处理预期的失败，但是成功的异常
           5、处理跳过
           6、处理其他Exception
          3、执行tearDown()
          4、self.doCleanups()，执行之前定义的清理函数
         result.stopTest(self)，result统计收尾工作
         result.stopTestRun()，如果这个方法在result中被重写(继承)，将在所有的测试执行完成后执行
        '''
    def debug(self):
        '''
         整个过程不收集信息
         执行流程：
          1、self.setUp()
          2、test_xxx()
          3、self.tearDown()
          4、while self._cleanups:
           doCleanups()
        '''
    ## 省略若干断言方法
```

### loader.py

> 负责加载各种标准方式的测试类，并将它们封装成TestSuite

全局函数：

```python
## 这3个函数，都是用于查找测试类or测试方法时，导入失败处理的函数
def _make_failed_import_test(name, suiteClass)
def _make_failed_load_tests(name, exception, suiteClass)
def _make_failed_test(classname, methodname, exception, suiteClass)
```

类：

```python
class TestLoader(object):
    '''
    类属性:
     testMethodPrefix: 测试方法名前缀，默认是test，所以修改这个，可以执行自定义开头的方法
     sortTestMehtodsUsing：测试方法排序使用的规则，默认是cmp
     suiteClass：指向suite.TestSuite类的引用
     _top_level_dir：None，在加载的过程中采用的是递归遍历，所以top_level用来判断当前目录
     是不是顶层
    ''' 
    
    def loadTestsFromTestCase(self, testCaseClass):
        ## 加载测试类中所有的单元测试，并将它们封装在TestSuite中返回
    def loadTestsFromModule(self, module, use_load_tests=True):
  ## 加载模块中所有的单元测试，最终返回一个列表，列表中就是一个个独立的TestSuite
    def loadTestsFromName(self, name, module=None):
  ## 加载指定路径下所有的单元测试
    def loadTestsFromNames(self, names, module=None):
  ## 加载指定路径下所有的单元测试，这里是Names，调用上面的方法完成的
    def getTestCaseNames(self, testCaseClass):
  ## 获取测试类中的单元测试方法名
    def discover(self, start_dir, pattern='test*.py', top_level_dir=None):
  ## 探索指定路径下，所有的py文件中的单元测试
    ## ……(省略若干查找细节)
```

### main.py

全局变量：

```python
FAILFAST     = "  -f, --failfast   Stop on first failure\n"
CATCHBREAK   = "  -c, --catch      Catch control-C and display results\n"
BUFFEROUTPUT = "  -b, --buffer     Buffer stdout and stderr during test runs\n"

USAGE_AS_MAIN = """\
Usage: %(progName)s [options] [tests]

Options:
  -h, --help       Show this message
  -v, --verbose    Verbose output
  -q, --quiet      Minimal output
%(failfast)s%(catchbreak)s%(buffer)s
Examples:
  %(progName)s test_module               - run tests from test_module
  %(progName)s module.TestClass          - run tests from module.TestClass
  %(progName)s module.Class.test_method  - run specified test method

[tests] can be a list of any number of test modules, classes and test
methods.

Alternative Usage: %(progName)s discover [options]

Options:
  -v, --verbose    Verbose output
%(failfast)s%(catchbreak)s%(buffer)s  -s directory     Directory to start discovery ('.' default)
  -p pattern       Pattern to match test files ('test*.py' default)
  -t directory     Top level directory of project (default to
                   start directory)

For test discovery all test modules must be importable from the top
level directory of the project.
"""

USAGE_FROM_MODULE = """\
Usage: %(progName)s [options] [test] [...]

Options:
  -h, --help       Show this message
  -v, --verbose    Verbose output
  -q, --quiet      Minimal output
%(failfast)s%(catchbreak)s%(buffer)s
Examples:
  %(progName)s                               - run default set of tests
  %(progName)s MyTestSuite                   - run suite 'MyTestSuite'
  %(progName)s MyTestCase.testSomething      - run MyTestCase.testSomething
  %(progName)s MyTestCase                    - run all 'test*' test methods
                                               in MyTestCase
"""
```

类：

```python
class TestProgram(object):
    def __init__(self, module='__main__', defaultTest=None, argv=None,
                    testRunner=None, testLoader=loader.defaultTestLoader,
                    exit=True, verbosity=1, failfast=None, catchbreak=None,
                    buffer=None)
     '''
     ……(一系列属性赋值)
        使用默认的Runner和Loader
        重点执行顺序
            self.parseArgs()，解析命令行参数
            self.runTests()，执行测试
     '''
        self.exit = exit
        self.failfast = failfast
        self.catchbreak = catchbreak
        self.verbosity = verbosity
        self.buffer = buffer
        self.defaultTest = defaultTest
        self.testRunner = testRunner
        self.testLoader = testLoader
        self.progName = os.path.basename(argv[0])
        self.parseArgs(argv)
        self.runTests()
        
    def usageExit(self, msg=None):
  ## 输出使用提示，并退出
 def parseArgs(self, argv):
        '''
        解析参数
        调用self._do_discovery()
        调用usageExit()
        调用createTests()
        '''
    def createTests(self):
        '''
        创建测试
        根据parseArgs解析出来的testNames，选择执行Loader下的loadTestsFromModule
        还是loadTestsFromNames
        '''
 def _do_discovery(self, argv, Loader=None):
        '''
        命令行参数中，有discover时触发，主要用于探索目录下的TestCase
  ……（细节不展开）
        '''
 def runTests(self):
        '''
        执行测试
        实例化一个runner对象，执行runner.run(self.test)
        self.test是createTests()生成出来的一个个独立的TestSuite
        '''
 

```

main.py文件中，还有重要的一句代码

```python
main = TestProgram ## 这里只是将类的引用传递给了main，不是实例化对象，注意
```

### result.py

全局函数：

```python
def failfast(method):
    '''
    本质是一个装饰器，根据名字，可以判断是用来发现测试过程中只要出现了Error就立即停止后续测试
    源码中使用频率低
    '''
```

类：

```python
class TestResult(object):
    '''
    类属性:
        _previousTestClass=None，记录上一个执行过的testCase的类，意义：在执行下一条
        testCase之前，执行上一条testCase的tearDown()函数
        _testRunEntered=False，记录当前testSuite是否在嵌套循环中，意义：执行时，都是以testSuite
        进行的，然后循环testSuite，将Suite中Case剥出来执行，如果Suite嵌套Suite，就会进入递归
        执行，所以此变量用于记录当前是否在嵌套中
        _moduleSetUpFailed=False，用于记录module级别的setUp是否执行失败，失败后module级别
        的tearDown并不会执行
    '''
    def __init__(self, stream=None, descriptions=None, verbosity=None)
        self.failfast=False     ## 默认关闭失败立即停止
        self.failures=[]     ## 统计失败个数
        self.errors=[]      ## 统计错误个数
        self.testRun=[]      ## 统计运行次数
        self.skipped=[]      ## 统计跳过次数
        self.expectedFailures=[]   ## 统计预期失败次数
        self.unexpectedSuccesses=[]   ## 统计非预期成功次数
        self.shouldStop=False    ## 默认不停止
        self.buffer=False     ## 默认没有buffer
        self._stdout_buffer=None   ## 默认没有buffer
        self._stderr_buffer=None   ## 默认没有buffer
        self._original_stdout=sys.stdout ## 默认输出为系统标准输出
        self._original_stderr=sys.stderr ## 默认错误输出为系统标准错误输出
        self._mirrorOutput=False   ## 默认不开启镜像输出
     def printErrors(self):
  ## 空方法，应该由继承类来实现
     def startTest(self, test):
  ## 执行Case前，统计次数，接管系统标准输出和标准错误输出
    def startTestRun(self):
        ## 空方法，应该由继承类实现，在startTest()之前执行
    def stopTest(self, test):
        ## 停止执行，接管系统标准输出和标准错误输出
    def stopTestRun(self):
        ## 空方法，应该由继承类实现，在stopTest()之后执行
 ## ……(省略若干addXX函数)
    ## addError/addFailure/addSuccess/addSkip/addExpectedFailure
    ## addUnexpectedSuccess
    def stop(self): 
        self.shouldStop = True
        ## 表示测试应该立即停止，这个方法不建议直接使用
```

### runner.py

类：

```python
class _WriteDecorator(object):
    ## 用于格式化信息
class TestTextResult(result.TestResult):
    ## 继承result.TestResult，用于保存测试记录
 ## 其中重写了父类里面的多个方法，startTest也被重写了(关注一下)
class TextTestRunner(object):
    ## 以文本形式显示结果的测试运行器类.当测试运行时，它打印出测试的名称和错误，并在测试运行结束时总结结果
    def __init__(self, stream=sys.stderr, descriptions=True, verbosity=1,
                 failfast=False, buffer=False, resultclass=None)
    '''
    stream：接收标准错误输出
 description
  result中用于描述TestCase
 verbosity
  详细输出。由级别控制
 failfast
  出现错误时，立即停止测试
 buffer
  执行期间，缓存标准输出和标准错误
 resultclass
  指明结果用什么result类管理
    '''
 def _makeResult(self):
  ## 内部方法，用于创建一个result对象
    def run(self, test):
        '''
        Run the given test case or test suite.（运行给定的Case或者Suite）
        注册Result、赋值failfast和buffer、设置开始时间
        判断result里面有没有startTestRun()，有没有都启动，没有就是一个空函数
        test(result):
            test，即传入的case或者suite对象，这时候调用的是case的__call__或者suite.__call__；
            suite的__call__方法是循环执行case.__call__；case.__call__执行的是对象中的run()；
            run(接收1个result参数)
        判断result里面有没有stopTestRun()，有没有都启动，没有就是一个空函数
        记录结束时间，计算执行时间，打印执行当中的error信息
        统计执行中预期的失败个数，不应该成功的个数，跳过个数
        整理成功信息、失败信息、错误信息、跳过信息写入result
        '''
```

### signals.py

类：

```python
class _InterruptHandler(object):
 ## 此类用于处理捕捉到Ctrl+C产生的异常
```

全局函数：

```python
def registerResult(result):
 ## 将result注册到一个弱引用的字典当中(可以学习一下什么是弱引用,import weakref)
def removeResult(result):
 ## 将result从弱引用字典中移出
def installHandler():
 ## 插桩，当系统捕获到指定信号量，用自定义的处理程序接管
def removeHandler(method=None):
 ## 将自定义的处理程序移出系统信号量的处理流程
```

### suite.py

全局函数：

```python
def _call_if_exists(parent, attr)
 ## 判断parent中是否有attr属性，有则执行，没有就赋值None
```

类：

```python
class BaseTestSuite(object)
 def __init__(self, tests=())
  self._tests=[]    ## 统计suite中有多少个case
  self.addTests(tests)  ## 将tests添加到self._tests中去
 def __iter__(self): return iter(self._tests)
  ## 实现了这个方法，表示实例化出来的对象。具有可迭代性
 def countTestCases(self)
  ## 统计suite中有多少个case
 def addTest(sefl, test)
  ## 判断test，是否有__call__属性，是否是TestCase的子类，是否是TestSuite的子类，
        ## 满足条件才会加入到self._tests列表中
 def addTests(self, tests)
  ## 循环调用self.addTest(self, test)
 def run(self, result)
  ## 循环suite中的case，并执行case自己的run方法
 def __call__(self, *args, **kwargs)
  ## 调用suite自己的run方法
 def debug(self)
  ## 没有result对象，不会收集errors等信息

class TestSuite(BaseTestSuite)
 def run(self, result, debug=False):
        '''
        1、根据result的类属性(_testRunEntered)，判断测试是否正“running”和
        是否为topLevel
        topLevel表示TestSuite对象第一次执行run()时的层级
  因为Suite里面可以套Suite和Case，所以有必要知道topLevel，而且topLevel存在的意思就是
  为了清理环境，这个层级的概念是建立在module级别的
  2、循环Suite，判断是Case还是Suite
  3、是Case，就先执行前一个Case的tearDownClass，再执行_handleModuleFixture
  (只需要执行一次，后续执行将会跳过)，执行当前Case的_handleClassSetUp()
  4、是Suite，就继续循环，然后执行run方法，有点递归的意思
  5、递归结束回到顶层时，执行前一个Case的tearDownClass和_handleModuleTearDown，
  同时将_testRunEntered置为False
        '''
 def debug(self):
  ## 不收集Error信息，执行Case
 def _handleClassSetUp(self, test, result):
        ## testFixture中执行class级别的setUp
 def _get_previous_module(self, result):
        ## testFixture中执行module级别的setUp和tearDown，被_handleModuleFixture调用
 def _handleModuleFixture(self, test, result):
        ## testFixture中处理module级别的setUp
 def _addClassOrModuleLevelException(self, result, exception, errorName):
        ## 当claas或者module在处理过程中出现了Exception，由此函数收集错误信息，用于输出
 def _handleModuleTearDown(self, result):
        ## testFixture中处理module级别的tearDown
 def _tearDownPreviousClass(self, test, result):
  ## 执行上一条testCase中class级别的tearDown()
        ## 如果有Failed将会执行返回
class _ErrorHolder(object):
 ## 用于在result给Error信息占位使用
```

### util.py

全局函数：

```python
def safe_repr(obj, short=False):
 ## 以安全的__repr__方式输出obj
def strclass(cls):
 ## 以字符串的形式输出类所在的模块和类的名字
def sorted_list_difference(expected, actual):
 ## 从expected和actual中，找出不同
def unorderable_list_difference(expected, actual, ignore_duplicate=False):
 ## 行为与sorted_list_difference类似
def _count_diff_all_purpose(actual, expected):
def _ordered_count(iterable):
def _count_diff_hashable(actual, expected):
```

## 程序入口

根据官网文档，unittest执行入口有以下几种方式：

```python
import unittest


## 方式一
if __name__ == '__main__':
    unittest.main()

## 方式二
## unittest 模块可以通过命令行运行模块、类和独立测试方法的测试:
python -m unittest test_module1 test_module2
python -m unittest test_module.TestClass
python -m unittest test_module.TestClass.test_method
```

## 命令行选项

- `-b`, --buffer

  在测试运行时，标准输出流与标准错误流会被放入缓冲区。成功的测试的运行时输出会被丢弃；测试不通过时，测试运行中的输出会正常显示，错误会被加入到测试失败信息。

- `-c`, --catch

  当测试正在运行时， Control-C 会等待当前测试完成，并在完成后报告已执行的测试的结果。当再次按下 Control-C 时，引发平常的 [`KeyboardInterrupt`](https://docs.python.org/zh-cn/2.7/library/exceptions.html#exceptions.KeyboardInterrupt) 异常。See [Signal Handling](https://docs.python.org/zh-cn/2.7/library/unittest.html#signal-handling) for the functions that provide this functionality.

- `-f`, --failfast

  当出现第一个错误或者失败时，停止运行测试。

## Test Discovery

探索性测试在 [`TestLoader.discover()`](https://docs.python.org/zh-cn/2.7/library/unittest.html#unittest.TestLoader.discover) 中实现，但也可以通过命令行使用。它在命令行中的基本用法如下：

```bash
cd project_directory
python -m unittest discover
```

`discover` 有以下选项：

- `-v`, --verbose

  更详细地输出结果。

- `-s`, --start-directory directory

  开始进行搜索的目录(默认值为当前目录 `.` )。

- `-p`, --pattern pattern

  用于匹配测试文件的模式（默认为 `test*.py` ）。

- `-t`, --top-level-directory directory

  指定项目的最上层目录（通常为开始时所在目录）。

`-s`，`-p` 和 `-t` 选项可以按顺序作为位置参数传入。以下两条命令是等价的：

```bash
python -m unittest discover -s project_directory -p "*_test.py"
python -m unittest discover project_directory "*_test.py"
```

正如可以传入路径那样，传入一个包名作为起始目录也是可行的，如 `myproject.subpackage.test` 。你提供的包名会被导入，它在文件系统中的位置会被作为起始目录。

```python
## python -m unittest discover
## 执行上面命令，也会主动的去执行单元测试，背后的逻辑如下
## 首先由于unittest目录下有__init__.py，表明这是一个包，然后还定义了__main__.py文件
## __main__.py文件的作用就是，可以将unittest这个包，当做一个文件去执行
'''
D:\学习总结\abcd>python -m unittest

----------------------------------------------------------------------
Ran 0 tests in 0.000s

OK

D:\学习总结\abcd>
'''
## 因此在命令中执行python -m unittest discover时，程序的入口在__main__.py文件中

## __main__.py
"""Main entry point"""

import sys
if sys.argv[0].endswith("__main__.py"):  
    ## sys.argv =['E:\\Software\\Python27\\lib\\unittest\\__main__.py', 'discover']
    ## 改造命令行参数的顺序，为了更好的识别sys.argv[1]是discover，以及sys.argv[2:]后的参数
    sys.argv[0] = "python -m unittest"

__unittest = True

from .main import main, TestProgram, USAGE_AS_MAIN
TestProgram.USAGE = USAGE_AS_MAIN

main(module=None)
## Note:这里虽然修改了sys.argv的顺序，但是main(module=None)并没有将sys.argv传进去
## 这是因为sys.argv已经被添加了python解释器的执行环境中的环境变量，类似一个全局变量

## main(module=None) ==> TestProgram(module=None)
## 进入到TestProgram.__init__方法中
## 进入到解析命令行参数时
class TestProgram(object):
    def __init__(self, ...):
        ## ...(省略中间代码)
        self.progName = os.path.basename(argv[0])
        self.parseArgs(argv)
        ## 执行已加载出来的testCase
        self.runTests()
    def parseArgs(self, argv):
        if len(argv) > 1 and argv[1].lower() == 'discover':
            ## 根据前面得知，len(argv) == 2,且argv[1] == 'discover'
            self._do_discovery(argv[2:])
            ## 此时加载出来的testCase(即Suite对象)已经赋值给了self.test
            return
        ## ...(省略中间代码)
    def _do_discovery(self, argv, Loader=None):
        if Loader is None:
            Loader = lambda: self.testLoader
        parser = optparse.OptionParser()
        ..## (创建一个命令行参数解析器，解析argv)
        options, args = parser.parse_args(argv)
        '''
        options：{'verbose': False, 'buffer': False,
                'pattern': 'test*.py', 'top': None,
                  'start': '.', 'catchbreak': False, 'failfast': False}
        args：[]
        '''
        ## 由于args是空列表，所以下面的for循环不会执行
        for name, value in zip(('start', 'pattern', 'top'), args):
            setattr(options, name, value)
  ## ...(省略中间代码)
        start_dir = options.start
        pattern = options.pattern
        top_level_dir = options.top
        loader = Loader()
        ## 参考下面的代码，self.test就是加载出来的suite对象
        self.test = loader.discover(start_dir, pattern, top_level_dir)

## loader.py
class TestLoader(object):
    def discover(self, start_dir, pattern='test*.py', top_level_dir=None):
        set_implicit_top = False
        if top_level_dir is None and self._top_level_dir is not None:
            ## make top_level_dir optional if called from load_tests in a package
            top_level_dir = self._top_level_dir
        ## 进入下面这个elif分支，因为top_level_dir==None(默认值)，
        ## self._top_level_dir初始值就是None
        elif top_level_dir is None:
            set_implicit_top = True
            top_level_dir = start_dir
        ## ...(省略中间代码)
        ## 标志位，标记目录是否可以import
        is_not_importable = False
        if os.path.isdir(os.path.abspath(start_dir)):
            ## 判断是否是目录
            start_dir = os.path.abspath(start_dir)
            ## 目录是否是顶层目录
            ## 这里start_dir == top_level_dir
            if start_dir != top_level_dir:
                ## 如果start_dir中没有__init__.py文件，可以佐证start_dir不是一个包
                is_not_importable = not os.path.isfile(
                    os.path.join(start_dir, '__init__.py'))
        else:
            ## support for discovery from dotted module names
            try:
                __import__(start_dir)
            except ImportError:
                is_not_importable = True
            else:
                the_module = sys.modules[start_dir]
                top_part = start_dir.split('.')[0]
                start_dir = os.path.abspath(os.path.dirname((the_module.__file__)))
                if set_implicit_top:
                    self._top_level_dir = self._get_directory_containing_module(top_part)
                    sys.path.remove(top_level_dir)
  ## is_not_importable == False
        if is_not_importable:
            raise ImportError('Start directory is not importable: %r' % start_dir)
        tests = list(self._find_tests(start_dir, pattern))
        ## self.suiteClass是suite.TestSuite的引用，加上()，相当于实例化了suite对象
        return self.suiteClass(tests)
    def _find_tests(self, start_dir, pattern):
        """Used by discovery. Yields test suites it loads."""
        ## 罗列出start_dir下所有的文件，目的就是为了找出单元所在的py文件
        paths = os.listdir(start_dir)
        ## 大循环遍历
        for path in paths:
            ## 获取绝对路径
            full_path = os.path.join(start_dir, path)
            if os.path.isfile(full_path):
                '''
                VALID_MODULE_NAME = re.compile(r'[_a-z]\w*\.py$', re.IGNORECASE)
                '''
                if not VALID_MODULE_NAME.match(path):  ## 匹配.py结尾的文件
                    ## valid Python identifiers only
                    continue
                '''
                self._match_path(self, path, full_path, pattern):
                 ## override this method to use alternative matching strategy
                 ## fnmatch用于匹配path中是否包含了pattern
                 return fnmatch(path, pattern)
                '''
                if not self._match_path(path, full_path, pattern): 
                    ## 经过前面的过滤，此处判断py文件名是否包含了pattern
                    continue
                ## if the test file matches, load it
                ## 获取full_path中的文件名，即xxx.py中的xxx
                name = self._get_name_from_path(full_path)
                try:
                    '''
                    self._get_module_from_name(self, name):
                     __import__(name)
                     return sys.modules[name]
                    '''
                    module = self._get_module_from_name(name)
                except:
                    yield _make_failed_import_test(name, self.suiteClass)
                else:
                    ## 下面的代码重点在于比较module.__file__名与绝对路径不带扩展名是否相等
                    mod_file = os.path.abspath(getattr(module, '__file__', full_path))
                    realpath = os.path.splitext(os.path.realpath(mod_file))[0]
                    fullpath_noext = os.path.splitext(os.path.realpath(full_path))[0]
                    if realpath.lower() != fullpath_noext.lower():
                        module_dir = os.path.dirname(realpath)
                        mod_name = os.path.splitext(os.path.basename(full_path))[0]
                        expected_dir = os.path.dirname(full_path)
                        msg = ("%r module incorrectly imported from %r. Expected %r. "
                               "Is this module globally installed?")
                        raise ImportError(msg % (mod_name, module_dir, expected_dir))
                    ## 见下面的代码
                    ## yield将加载出来的suite对象返给调用者
                    yield self.loadTestsFromModule(module)
            elif os.path.isdir(full_path):
                if not os.path.isfile(os.path.join(full_path, '__init__.py')):
                    continue

                load_tests = None
                tests = None
                if fnmatch(path, pattern):
                    ## only check load_tests if the package directory itself matches the filter
                    name = self._get_name_from_path(full_path)
                    package = self._get_module_from_name(name)
                    load_tests = getattr(package, 'load_tests', None)
                    tests = self.loadTestsFromModule(package, use_load_tests=False)

                if load_tests is None:
                    if tests is not None:
                        ## tests loaded from package file
                        yield tests
                    ## recurse into the package
                    for test in self._find_tests(full_path, pattern):
                        yield test
                else:
                    try:
                        yield load_tests(self, tests, pattern)
                    except Exception, e:
                        yield _make_failed_load_tests(package.__name__, e,
                                                      self.suiteClass)
 def loadTestsFromModule(self, module, use_load_tests=True):
        """Return a suite of all test cases contained in the given module"""
        tests = []
        for name in dir(module):
            ## dir(module)遍历module中所有的属性、类、函数，并用getattr读出来，
            obj = getattr(module, name)
            '''
            Python 2.7.14 
            Type "help", "copyright", "credits" or "license" for more information.
            >>> type(object)
            <type 'type'>
            >>>
            判断obj是不是一个对象，和obj是不是TestCase的子类
            '''
            if isinstance(obj, type) and issubclass(obj, case.TestCase):
                tests.append(self.loadTestsFromTestCase(obj))
  ## module没有’load_tests‘属性
        load_tests = getattr(module, 'load_tests', None)
        ## 将读出来的tests实例化成为一个suite对象
        ## tests列表中实际上是一个个suite对象，类似于
        ## tests = [suite1, suite2, suite3, ...]
        ## suite对象允许嵌套suite对象
        tests = self.suiteClass(tests)
        ## 不会进入下面分支
        if use_load_tests and load_tests is not None:
            try:
                return load_tests(self, tests, None)
            except Exception, e:
                return _make_failed_load_tests(module.__name__, e,
                                               self.suiteClass)
        ## 此处返回的是suite对象，包含了module下所有的case
        return tests

    def loadTestsFromTestCase(self, testCaseClass):
        """Return a suite of all test cases contained in testCaseClass"""
        ## testCaseClass是编写的单元测试中，测试类的引用
        ## 判断是不是suite和case的子类
        if issubclass(testCaseClass, suite.TestSuite):
            raise TypeError("Test cases should not be derived from TestSuite." \
                                " Maybe you meant to derive from TestCase?")
        ## 获取case的名字(列表)
        testCaseNames = self.getTestCaseNames(testCaseClass)
        if not testCaseNames and hasattr(testCaseClass, 'runTest'):
            testCaseNames = ['runTest']
        ## 利用map函数，将testCaseClass与testCaseNames进行实例化
        ## map函数将testCaseNames实例化为TestCase对象
        ## self.suiteClass([TestCase对象1, TestCase对象2, ...])实例化为一个suite对象
        loaded_suite = self.suiteClass(map(testCaseClass, testCaseNames))
        return loaded_suite

    def getTestCaseNames(self, testCaseClass):
        """Return a sorted sequence of method names found within testCaseClass
        """
        def isTestMethod(attrname, testCaseClass=testCaseClass,
                         prefix=self.testMethodPrefix):
            ## attrname是testCaseClass中的属性(即编写的单元测试类中属性，如：__call__/__class__等)
            ## self.testMethodPrefix实际上是 "test"
            ## 这里的判断，就是要取出test开头,且这个属性还具有__call__属性
            ## 方法和函数都是具有__call__属性的，不然不能被调用
            return attrname.startswith(prefix) and \
                hasattr(getattr(testCaseClass, attrname), '__call__')
        ## 利用filter过滤出testCaseClass中，需要的方法
        testFnNames = filter(isTestMethod, dir(testCaseClass))
        ## 这里是将testFnNames进行排序
        if self.sortTestMethodsUsing:
            testFnNames.sort(key=_CmpToKey(self.sortTestMethodsUsing))
        ## 返回的是一个列表，包含了所有以test开头的方法名，注意是方法名，不是方法的引用
        return testFnNames
```

## 执行方式

```python
import unittest 

class UCTestCase(unittest.TestCase):
    def setUp(self):
        #测试前需执行的操作  
    def tearDown(self):
        #测试用例执行完后所需执行的操作
    ## 测试用例1
    def testCreateFolder(self):
        #具体的测试脚本
    ## 测试用例2
    def testDeleteFolder(self):
        #具体的测试脚本

if __name__ == "__main__":
    ## 方式1，直接通过main()执行
    unittest.main()
    
    ## 方式2
    ## 构造测试集，只有单个suite
    suite = unittest.TestSuite()
    suite.addTest(UC7TestCase("testCreateFolder"))
    suite.addTest(UC7TestCase("testDeleteFolder")) 
    ## 执行测试
    runner = unittest.TextTestRunner()
    runner.run(suite)
    
    ## 方式3
    #此用法可以同时测试多个类，用多个suite构造
    suite1 = unittest.TestLoader().loadTestsFromTestCase(TestCase1) 
    suite2 = unittest.TestLoader().loadTestsFromTestCase(TestCase2) 
    ## 实质上还是suite嵌套suite
    suite = unittest.TestSuite([suite1, suite2]) 
    unittest.TextTestRunner(verbosity=2).run(suite)
    
    ## 实际上都是利用的runner来执行
```
