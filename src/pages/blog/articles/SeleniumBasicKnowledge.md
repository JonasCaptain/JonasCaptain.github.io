---
layout: "@layouts/ArticleLayout.astro"
title: Selenium学习系列----基础知识+常规操作
description: Selenium学习系列----基础知识+常规操作
date: 2024-12-19 21:15:42
tags:
    - selenium
---

> 本文章摘抄至本人CSDN文章

# 基础知识
selenium官方地址：https://www.selenium.dev/
先思考一个问题：为什么绝大部分博主在学习一项技术时，都会推荐官网，官网到底能带给你什么？
答：首先，官网的资料一定的是最官方、最权威的、最新的、最全面的。基本上你想实现的操作，都能在官网上找到对应的文档。不过官方文档有一个不友好的地方，那就是几乎全是英文，语言能力弱的我，每次都需要靠翻译软件摸索着过河。但是，摸索着过河比站在原地踏步要强很多，所以希望看见这篇博文的各位，能从官网或者官方渠道去寻求帮助。

## 概念知晓
不对名词深究，只简单知晓其含义、作用即可，需要使用时再回过头仔细研究。
Selenium WebDriver：通过浏览器的驱动程序去控制浏览器，以实现期望的动作，能完成控制浏览器实现操作的这个东东，就是WebDriver
Selenium IDE：专门为编写Selenium代码而制作的一款集成开发工具(IDE)，主要是通过屏幕录制，记录操作，最终导出指定编程语言的代码。
Selenium Grid：这是为了分布式执行Selenium而准备，本系列文章暂不涉及，如果有时间会弥补这块内容。

# 准备工作
`本文以Python语言贯穿整篇博文；selenium官网文档中部分内容未涉及到python示例代码，将采用Python语言补充`
## 1. 安装类库
```python
pip install selenium
# 上面这种方式将直接安装最新版本
# 如果为了学习，建议使用最新版本
# 如果为了工作，建议采用稳定版本；
# 稳定版本一般是工作中已经正在使用的版，或者是已知版本中Bug较少的版本；
pip install selenium==3.0.0
# 这种方式将安装指定版本的selenium类库
# 当然也有其他安装方式，不过还是建议使用pip
```
## 2. 安装Driver
官网文档：https://www.selenium.dev/zh-cn/documentation/webdriver/getting_started/install_drivers/
现在官网文档支持中文显示
这里只简述一下3种方式：
1. 驱动管理
大多数机器会自动更新浏览器, 但驱动程序不会. 为了确保为浏览器提供正确的驱动程序, 这里有许多第三方库可为您提供帮助.
	```python
	pip install webdriver-manager
	# 使用之前先安装
	from selenium import webdriver
	from webdriver_manager.chrome import ChromeDriverManager
	from selenium.webdriver.chrome.service import Service
	
	service = Service(executable_path=ChromeDriverManager().install())
	driver = webdriver.Chrome(service=service)
	```
	这里仅仅是官网提供的简单示例，更多更全的例子，在这里可以参考：https://github.com/SergeyPirogov/webdriver_manager
2. 配置PATH环境变量
	首先得手动下载驱动，驱动与浏览器版本是相关联的，必须匹配上才能使驱动生效。
	这种方式网上的教程比比皆是，类似`JAVAHOME`，一定要放在系统PATH里面，不要放在User下面的PATH，否则会报错找不到。
3. 硬编码
	即将驱动的位置，写在代码里面，执行代码时，将从指定路径下获取Driver。
	```python
	service = Service(executable_path="/path/to/chromedriver")
	# 或者
	service = Service(executable_path="D:\\chormedriver.exe")
	driver = webdriver.Chrome(service=service)
	```
	硬编码的方式，适合写代码、调试的时候。
## 3. Open、Close Browser
```python
from selenium import webdriver
from selenium.webdriver.chrome.options as ChromeOptions

# 这里是浏览器对应的配置类
options = ChromeOptions()
# 添加配置的方法如下:
options.binary_location = "D:\\Chrome\\Chrome.exe"
options.add_argument("--headless")
options.add_argument("--disable-gpu")
# 实例化浏览器对象时，将配置类传入即可
driver = webdriver.Chrome(options=options)
# 打开浏览器
driver.get("https://www.google.com")
# driver.close()仅仅是关闭浏览器，但不会退出进程
# 关闭浏览器并结束整个进程
driver.quit()
```
```python
'''
原理简述：
	options: 这是一个关于浏览器配置的对象，而这个selenium.webdriver.chrome.options
			 继承的是selenium.webdriver.chromium.options 所以关于options更多的属性
			 得去找父类.
			 在这里本人只讲几个简单的，或者使用过的，没有使用过或者没有深入研究过不敢造
			 次.
			 binary_location,这个属性用于指定浏览器的绝对路径，适用于Chrome.exe浏览器
			 启动程序不在默认路径,Chrome默认安装在C盘，如果你的不在C盘需要指定一下.
			 headless,无头模式，即不启动浏览器窗口,但是会有浏览器进程产生.通常启动无头
			 模式，都是这种方式：options.add_argument('--headless')，这是推荐写法。
			 当然headless本身已经是一个property，可以直接对其赋值options.headless = True
			 此外headless还有个搭配对象,options.add_argument('--disable-gpu');
			 当然--disable-gpu这个选项也是针对浏览器的，有的支持，有的不支持，需要查看
			 浏览器对应的options文件。
	driver.get()用于打开一个网页，这里需要注意打开的网页，必须携带http://协议头，否则
			 会出错，遇到了可以自行排查。
	driver.quit()关于浏览器并结束会话。driver.close()仅仅是关闭浏览器，但是通过
			 selenium创建出来的会话并没有关闭，会残留进程。
'''
```
# 操作串讲
## 1.  driver都支持哪些操作？
`后面有更详细的分类`
- 获取浏览器title  driver.title 
title是一个属性，直接读取，是RemoteWebdriver的一个属性。所有浏览器都是继承于它
- 查找元素 driver.find_element(self, by=By.ID, value=None) 
这个是当前推荐的使用方法，用于定位一个元素。定位的手段是通过元素ID，Value即是元素的ID值。当然以前使用的find_element_by_id还是可以用，不过会有<strong>warning</strong>提示
- 查找元素集 driver.find_elements(self, by=By.ID, value=None)
这个是当前推荐的使用方法，用于定位一个元素。定位的手段是通过元素ID，Value即是元素的ID值。当然以前使用的find_elements_by_id还是可以用，不过会有<strong>warning</strong>提示
- 截图 driver.save_screenshot(self,filename) 
filename即是保存图片的绝对路径。实际使用经验较少
- 获取当前窗口的URL driver.current_url
current_url是driver的一个属性，直接使用
- 打开网站 driver.get("URL")
打开指定网址
- 浏览器后退按钮 driver.back()
- 浏览器前进按钮 driver.forward()
- 浏览器刷新按钮 driver.refresh()
- 
除此之外还有一些调整浏览器窗口大小的方法，比较重要的一些方法将在后面涉及，比如driver.execute_script()和driver.execute_async_script()。

## 2. 弹窗处理
网页上一般有3中类型的弹窗，它们往往都是起一个提示的作用，根据提示程度的不同分为警告框、确认框和提示框。
- 警告框 Alert
```python
# Click the link to activate the alert
driver.find_element(By.LINK_TEXT, "See an example alert").click()

# Wait for the alert to be displayed and store it in a variable
alert = wait.until(expected_conditions.alert_is_present())

# Store the alert text in a variable
text = alert.text

# Press the OK button
alert.accept()
```
- 确认框 Confirm
```python
# Click the link to activate the alert
driver.find_element(By.LINK_TEXT, "See a sample confirm").click()

# Wait for the alert to be displayed
wait.until(expected_conditions.alert_is_present())

# Store the alert in a variable for reuse
alert = driver.switch_to.alert

# Store the alert text in a variable
text = alert.text

# Press the Cancel button
alert.dismiss()
```
- 提示框 Prompt
```python
# Click the link to activate the alert
driver.find_element(By.LINK_TEXT, "See a sample prompt").click()

# Wait for the alert to be displayed
wait.until(expected_conditions.alert_is_present())

# Store the alert in a variable for reuse
alert = Alert(driver)

# Type your message
alert.send_keys("Selenium")

# Press the OK button
alert.accept()
```
## 3. Cookies处理
Cookie主要用来区分身份信息或者登录状态。
 - 添加 Cookie
这个方法常常用于将cookie添加到当前访问的上下文中. 添加Cookie仅接受一组已定义的可序列化JSON对象. 这里 是一个链接, 用于描述可接受的JSON键值的列表

首先, 您需要位于有效Cookie的域上. 如果您在开始与网站进行交互之前尝试预设cookie, 并且您的首页很大或需要一段时间才能加载完毕, 则可以选择在网站上找到一个较小的页面 (通常404页很小, 例如 http://example.com/some404page)
```python
from selenium import webdriver

driver = webdriver.Chrome()

driver.get("http://www.example.com")

# Adds the cookie into current browser context
driver.add_cookie({"name": "key", "value": "value"})
```
- 获取命名的 Cookie
此方法返回与cookie名称匹配的序列化cookie数据中所有关联的cookie.
```python
from selenium import webdriver

driver = webdriver.Chrome()

# Navigate to url
driver.get("http://www.example.com")

# Adds the cookie into current browser context
driver.add_cookie({"name": "foo", "value": "bar"})

# Get cookie details with named cookie 'foo'
print(driver.get_cookie("foo"))
```
- 获取全部 Cookies
此方法会针对当前访问上下文返回“成功的序列化cookie数据”. 如果浏览器不再可用, 则返回错误.
```python
from selenium import webdriver

driver = webdriver.Chrome()

# Navigate to url
driver.get("http://www.example.com")

driver.add_cookie({"name": "test1", "value": "cookie1"})
driver.add_cookie({"name": "test2", "value": "cookie2"})

# Get all available cookies
print(driver.get_cookies())
```
- 删除 Cookie
此方法删除与提供的cookie名称匹配的cookie数据.
```python
from selenium import webdriver
driver = webdriver.Chrome()

# Navigate to url
driver.get("http://www.example.com")
driver.add_cookie({"name": "test1", "value": "cookie1"})
driver.add_cookie({"name": "test2", "value": "cookie2"})

# Delete a cookie with name 'test1'
driver.delete_cookie("test1")
```
- 删除所有 Cookies
此方法删除当前访问上下文的所有cookie.
```python
from selenium import webdriver
driver = webdriver.Chrome()

# Navigate to url
driver.get("http://www.example.com")
driver.add_cookie({"name": "test1", "value": "cookie1"})
driver.add_cookie({"name": "test2", "value": "cookie2"})

#  Deletes all cookies
driver.delete_all_cookies()
```
- Same-Site Cookie属性
此属性允许用户引导浏览器控制cookie, 是否与第三方站点发起的请求一起发送. 引入其是为了防止CSRF（跨站请求伪造）攻击.
Same-Site cookie属性接受以下两种参数作为指令
**Strict**:
当sameSite属性设置为 Strict, cookie不会与来自第三方网站的请求一起发送.
**Lax**:
当您将cookie sameSite属性设置为 Lax, cookie将与第三方网站发起的GET请求一起发送.
注意: 到目前为止, 此功能已在Chrome(80+版本), Firefox(79+版本)中提供, 并适用于Selenium 4以及更高版本.
```python
from selenium import webdriver

driver = webdriver.Chrome()

driver.get("http://www.example.com")
# Adds the cookie into current browser context with sameSite 'Strict' (or) 'Lax'
driver.add_cookie({"name": "foo", "value": "value", 'sameSite': 'Strict'})
driver.add_cookie({"name": "foo1", "value": "value", 'sameSite': 'Lax'})
cookie1 = driver.get_cookie('foo')
cookie2 = driver.get_cookie('foo1')
print(cookie1)
print(cookie2)
```
## 4. iframes处理
iframes是一种现在已被弃用的方法，用于从同一域中的多个文档构建站点布局。除非你使用的是 HTML5 之前的 webapp，否则你不太可能与他们合作。内嵌框架允许插入来自完全不同领域的文档，并且仍然经常使用。
如果您需要使用框架或 iframe, WebDriver 允许您以相同的方式使用它们。考虑 iframe 中的一个按钮。 如果我们使用浏览器开发工具检查元素，我们可能会看到以下内容:
```html
<div id="modal">
  <iframe id="buttonframe"name="myframe"src="https://seleniumhq.github.io">
   <button>Click here</button>
 </iframe>
</div>
```
如果不是 iframe，我们可能会使用如下方式点击按钮:
```python
# 这不会工作
driver.find_element(By.TAG_NAME, 'button').click()
```
但是，如果 iframe 之外没有按钮，那么您可能会得到一个 no such element 无此元素 的错误。 这是因为 Selenium 只知道顶层文档中的元素。为了与按钮进行交互，我们需要首先切换到框架， 这与切换窗口的方式类似。WebDriver 提供了三种切换到iframe的方法。
使用 WebElement 进行切换是最灵活的选择。您可以使用首选的选择器找到框架并切换到它。
```python
# 存储网页元素
iframe = driver.find_element(By.CSS_SELECTOR, "#modal > iframe")
# 切换到选择的 iframe
driver.switch_to.frame(iframe)
# 单击按钮
driver.find_element(By.TAG_NAME, 'button').click()
```
如果您的 frame 或 iframe 具有 id 或 name 属性，则可以使用该属性。如果名称或 id 在页面上不是唯一的， 那么将切换到找到的第一个。
```python
# 通过 id 切换框架
driver.switch_to.frame('buttonframe')
# 单击按钮
driver.find_element(By.TAG_NAME, 'button').click()
```
还可以使用frame的索引， 例如可以使用JavaScript中的 window.frames 进行查询.
```python
# 基于索引切换到第 2 个 iframe
iframe = driver.find_elements_by_tag_name('iframe')[1]
# 切换到选择的 iframe
driver.switch_to.frame(iframe)
```
离开 iframe 或 frameset，切换回默认内容，如下所示:
```python
# 切回到默认内容
driver.switch_to.default_content()
```
## 5. 窗口与标签
窗口和标签页
WebDriver 没有区分窗口和标签页。如果你的站点打开了一个新标签页或窗口，Selenium 将允许您使用窗口句柄来处理它。 每个窗口都有一个唯一的标识符，该标识符在单个会话中保持持久性。你可以使用以下方法获得当前窗口的窗口句柄:
```python
driver.current_window_handle
```
切换窗口或标签页
单击在 <a href=“https://seleniumhq.github.io"target="_blank”>新窗口 中打开链接， 则屏幕会聚焦在新窗口或新标签页上，但 WebDriver 不知道操作系统认为哪个窗口是活动的。 要使用新窗口，您需要切换到它。 如果只有两个选项卡或窗口被打开，并且你知道从哪个窗口开始， 则你可以遍历 WebDriver， 通过排除法可以看到两个窗口或选项卡，然后切换到你需要的窗口或选项卡。
不过，Selenium 4 提供了一个新的 api NewWindow 它创建一个新选项卡 (或) 新窗口并自动切换到它。
```python
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

# 启动驱动程序
with webdriver.Firefox() as driver:
    # 打开网址
    driver.get("https://seleniumhq.github.io")
    # 设置等待
    wait = WebDriverWait(driver, 10)
    # 存储原始窗口的 ID
    original_window = driver.current_window_handle
    # 检查一下，我们还没有打开其他的窗口
    assert len(driver.window_handles) == 1
    # 单击在新窗口中打开的链接
    driver.find_element(By.LINK_TEXT, "new window").click()
    # 等待新窗口或标签页
    wait.until(EC.number_of_windows_to_be(2))
    # 循环执行，直到找到一个新的窗口句柄
    for window_handle in driver.window_handles:
        if window_handle != original_window:
            driver.switch_to.window(window_handle)
            break
    # 等待新标签页完成加载内容
    wait.until(EC.title_is("SeleniumHQ Browser Automation"))
```
创建新窗口(或)新标签页并且切换
创建一个新窗口 (或) 标签页，屏幕焦点将聚焦在新窗口或标签在上。您不需要切换到新窗口 (或) 标签页。如果除了新窗口之外， 您打开了两个以上的窗口 (或) 标签页，您可以通过遍历 WebDriver 看到两个窗口或选项卡，并切换到非原始窗口。
注意: 该特性适用于 Selenium 4 及其后续版本。
```python
# 打开新标签页并切换到新标签页
driver.switch_to.new_window('tab')
# 打开一个新窗口并切换到新窗口
driver.switch_to.new_window('window')
```
关闭窗口或标签页
当你完成了一个窗口或标签页的工作时，_并且_它不是浏览器中最后一个打开的窗口或标签页时，你应该关闭它并切换回你之前使用的窗口。 假设您遵循了前一节中的代码示例，您将把前一个窗口句柄存储在一个变量中。把这些放在一起，你会得到:
```python
#关闭标签页或窗口
driver.close()
#切回到之前的标签页或窗口
driver.switch_to.window(original_window)
```
如果在关闭一个窗口后忘记切换回另一个窗口句柄，WebDriver 将在当前关闭的页面上执行，并触发一个 No Such Window Exception 无此窗口异常。必须切换回有效的窗口句柄才能继续执行。

在会话结束时退出浏览器
当你完成了浏览器会话，你应该调用 quit 退出，而不是 close 关闭:
```python
driver.quit()
```
退出将会
 - 关闭所有与 WebDriver 会话相关的窗口和选项卡
 - 结束浏览器进程
 - 结束后台驱动进程
 - 通知 Selenium Grid 浏览器不再使用，以便可以由另一个会话使用它(如果您正在使用 Selenium Grid)

调用 quit() 失败将留下额外的后台进程和端口运行在机器上，这可能在以后导致一些问题。
有的测试框架提供了一些方法和注释，您可以在测试结束时放入 teardown() 方法中。
```python
# unittest teardown
# https://docs.python.org/3/library/unittest.html?highlight=teardown#unittest.TestCase.tearDown
def tearDown(self):
	self.driver.quit()
```
如果不在测试上下文中运行 WebDriver，您可以考虑使用 try / finally，这是大多数语言都提供的， 这样一个异常处理仍然可以清理 WebDriver 会话。
```python
try:
    #WebDriver 代码…
finally:
	driver.quit()
```
Python 的 WebDriver 现在支持 Python 上下文管理器，当使用 with 关键字时，可以在执行结束时自动退出驱动程序。
```python
with webdriver.Firefox() as driver:
  # WebDriver 代码…

# 在此缩进位置后 WebDriver 会自动退出
```
窗口管理
屏幕分辨率会影响 web 应用程序的呈现方式，因此 WebDriver 提供了移动和调整浏览器窗口大小的机制。

获取窗口大小
获取浏览器窗口的大小(以像素为单位)。
```python
# 分别获取每个尺寸
width = driver.get_window_size().get("width")
height = driver.get_window_size().get("height")

# 或者存储尺寸并在以后查询它们
size = driver.get_window_size()
width1 = size.get("width")
height1 = size.get("height")
```
设置窗口大小
恢复窗口并设置窗口大小。
```python
driver.set_window_size(1024, 768)
```
得到窗口的位置
获取浏览器窗口左上角的坐标。
```python
# 分别获取每个尺寸
x = driver.get_window_position().get('x')
y = driver.get_window_position().get('y')

# 或者存储尺寸并在以后查询它们
position = driver.get_window_position()
x1 = position.get('x')
y1 = position.get('y')
```
设置窗口位置
将窗口移动到设定的位置。
```python
# 将窗口移动到主显示器的左上角
driver.set_window_position(0, 0)
```
最大化窗口
扩大窗口。对于大多数操作系统，窗口将填满屏幕，而不会阻挡操作系统自己的菜单和工具栏。
```python
driver.maximize_window()
```
最小化窗口
最小化当前浏览上下文的窗口. 这种命令的精准行为将作用于各个特定的窗口管理器.

最小化窗口通常将窗口隐藏在系统托盘中.
注意: 此功能适用于Selenium 4以及更高版本.
```python
driver.minimize_window()
```
全屏窗口
填充整个屏幕，类似于在大多数浏览器中按下 F11。
```python
driver.fullscreen_window()
```
屏幕截图
用于捕获当前浏览上下文的屏幕截图. WebDriver端点 屏幕截图 返回以Base64格式编码的屏幕截图.
```python
from selenium import webdriver

driver = webdriver.Chrome()

# Navigate to url
driver.get("http://www.example.com")

# Returns and base64 encoded string into image
driver.save_screenshot('./image.png')

driver.quit()
```
元素屏幕截图
用于捕获当前浏览上下文的元素的屏幕截图. WebDriver端点 屏幕截图 返回以Base64格式编码的屏幕截图.
```python
from selenium import webdriver
from selenium.webdriver.common.by import By

driver = webdriver.Chrome()

# Navigate to url
driver.get("http://www.example.com")

ele = driver.find_element(By.CSS_SELECTOR, 'h1')

# Returns and base64 encoded string into image
ele.screenshot('./image.png')

driver.quit()
```
执行脚本
在当前frame或者窗口的上下文中，执行JavaScript代码片段.
```python
# Stores the header element
header = driver.find_element(By.CSS_SELECTOR, "h1")

# Executing JavaScript to capture innerText of header element
driver.execute_script('return arguments[0].innerText', header)
```
打印页面
打印当前浏览器内的页面

注意: 此功能需要无头模式下的Chromium浏览器
```python
from selenium.webdriver.common.print_page_options import PrintOptions

print_options = PrintOptions()
print_options.page_ranges = ['1-2']

driver.get("printPage.html")

base64code = driver.print_page(print_options)
```
## 6. 元素交互

用于操纵表单的高级指令集.
仅有五种基本命令可用于元素的操作:

- 点击 (适用于任何元素)
- 发送键位 (仅适用于文本字段和内容可编辑元素)
- 清除 (仅适用于文本字段和内容可编辑元素)
- 提交 (仅适用于表单元素)
- 选择 (参见 选择列表元素)

### 附加验证
这些方法的设计目的是尽量模拟用户体验, 所以, 与 Actions接口 不同, 在指定制定操作之前, 会尝试执行两件事.
1. 如果它确定元素在视口之外, 则会将元素滚动到视图中, 特别是将元素底部与视口底部对齐.
2. 确保元素在执行操作之前是可交互的 . 这可能意味着滚动不成功, 或者该元素没有以其他方式显示.
确定某个元素是否显示在页面上太难了 无法直接在webdriver规范中定义, 因此Selenium发送一个带有JavaScript原子的执行命令, 检查是否有可能阻止该元素显示. 如果确定某个元素不在视口中, 不显示, 不可 键盘交互, 或不可 指针交互, 则返回一个元素不可交互 错误.

点击
元素点击命令 执行在 元素中央. 如果元素中央由于某些原因被 遮挡 , Selenium将返回一个 元素点击中断 错误.

发送键位
元素发送键位命令 将录入提供的键位到 可编辑的 元素. 通常, 这意味着元素是具有 文本 类型的表单的输入元素或具有 内容可编辑 属性的元素. 如果不可编辑, 则返回 无效元素状态 错误.

以下 是WebDriver支持的按键列表.
```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
driver = webdriver.Firefox()

# Navigate to url
driver.get("http://www.google.com")

# Enter "webdriver" text and perform "ENTER" keyboard action
driver.find_element(By.NAME, "q").send_keys("webdriver" + Keys.ENTER)
```
清除
元素清除命令 重置元素的内容. 这要求元素 可编辑, 且 可重置. 通常, 这意味着元素是具有 文本 类型的表单的输入元素或具有 内容可编辑 属性的元素. 如果不满足这些条件, 将返回 无效元素状态 错误.
```python
from selenium import webdriver
from selenium.webdriver.common.by import By
driver = webdriver.Chrome()

# Navigate to url
driver.get("http://www.google.com")
# Store 'SearchInput' element
SearchInput = driver.find_element(By.NAME, "q")
SearchInput.send_keys("selenium")
# Clears the entered text
SearchInput.clear()
```
**<font color="#FF0000">提交</font>**
在Selenium 4中, 不再通过单独的端点以及脚本执行的方法来实现. 因此, 建议不要使用此方法, 而是单击相应的表单提交按钮.

# 元素定位串讲
## 查询元素
根据提供的定位值定位元素.
使用selenium的一个最基本的方面是获取元素参考。Selenium提供了许多内置的定位策略来唯一地识别元素。在非常高级的场景中，有很多方法可以使用定位器。为了这个文档的目的，让我们考虑一下这个HTML片段：
```html
<ol id="vegetables">
 <li class="potatoes">…
 <li class="onions">…
 <li class="tomatoes"><span>Tomato is a Vegetable</span>…
</ol>
<ul id="fruits">
  <li class="bananas">…
  <li class="apples">…
  <li class="tomatoes"><span>Tomato is a Fruit</span>…
</ul>
```
### 单元素匹配find_element
在selenium中大部分的元素方式将匹配页面上的多个元素。元素除非具有唯一性(比如ID)，否则find_element方法将返回对给定元素结果集中找到的第一个元素的引用。
首先selenium会评估整个DOM
在driver上调用find element方法时，它返回对DOM中与提供的定位器匹配的第一个元素的引用。该值可以存储并用于将来的元素操作。在上面的HTML示例中，有两个元素的类名为“tomatoes”，因此该方法将返回“vegetables”列表中的元素。
```python
vegetable = driver.find_element(By.CLASS_NAME, "tomatoes")
```
评估DOM的子集
与其在整个DOM中找到唯一的定位器，不如将搜索范围缩小到另一个定位元素的范围。在上面的示例中，有两个元素的类名为“tomatoes”，而获取第二个元素的引用则更具挑战性。
一种解决方案是定位具有唯一属性的元素，该属性是所需元素的父类，然后在该对象上调用find element：
```python
fruits = driver.find_element(By.ID, "fruits")
fruit = fruits.find_elements_by_id("tomatoes") 
```
优化定位器
嵌套定位元素可能不是最佳的定位策略，因为它需要向浏览器发出两个单独的命令。
为了稍微提高性能，我们可以使用CSS或XPath在单个命令中查找该元素。
在本例中，我们将使用CSS选择器：
```python
fruit = driver.find_element_by_css_selector("#fruits .tomatoes")
```
### 多元素匹配find_elements
find_elements方法返回元素引用的集合。如果没有匹配到结果，则返回一个空列表。下面这个例子中，所有的fruit和vegetable列表项的引用将在集合中返回。
```python
plants = driver.find_elements(By.TAG_NAME, "li")
 ```
 获取元素
得到一个元素集合后，需要使用特定的元素，需要迭代该集合并确定所需的元素。
```python
from selenium import webdriver
from selenium.webdriver.common.by import By

driver = webdriver.Firefox()

# Navigate to Url
driver.get("https://www.example.com")

# Get all the elements available with tag name 'p'
elements = driver.find_elements(By.TAG_NAME, 'p')

for e in elements:
    print(e.text)
```
从元素中查找元素
主要用于在父元素的上下文中查找匹配的子元素列表(WebElement)。为了实现这一点，父元素可以通过find_elements方法直接访问子元素。
```python
from selenium import webdriver
from selenium.webdriver.common.by import By

driver = webdriver.Chrome()
driver.get("https://www.example.com")

# Get element with tag name 'div'
element = driver.find_element(By.TAG_NAME, 'div')

# Get all the elements available with tag name 'p'
elements = element.find_elements(By.TAG_NAME, 'p')
for e in elements:
    print(e.text)
```
获取活动元素
它用于跟踪（或）查找在当前浏览上下文中具有焦点的DOM元素。
```python
from selenium import webdriver
from selenium.webdriver.common.by import By

driver = webdriver.Chrome()
driver.get("https://www.google.com")
driver.find_element(By.CSS_SELECTOR, '[name="q"]').send_keys("webElement")
# Get attribute of current active element
attr = driver.switch_to.active_element.get_attribute("title")
print(attr)
```
##  元素信息
判断元素是否显示 is_displayed
此方法用于检查连接的元素是否显示在网页上。返回一个布尔值，如果连接的元素显示在当前浏览上下文中，则返回True，否则返回false。
这个功能在w3c规范中有提到，但没有定义，因为不可能涵盖所有潜在的条件。因此，Selenium不能期望驱动程序直接实现这个功能，而是依赖于直接执行一个大型JavaScript函数。这个函数对树中的元素的性质和关系进行许多近似，以返回一个值。

判断元素是否被选中  is_selected
此方法确定是否 已选择 引用的元素. 此方法广泛用于复选框, 单选按钮, 输入元素和选项元素.

返回一个布尔值, 如果在当前浏览上下文中 已选择 引用的元素, 则返回 True, 否则返回 False.
```python
# Navigate to url
driver.get("https://the-internet.herokuapp.com/checkboxes")

# Returns true if element is checked else returns false
value = driver.find_element(By.CSS_SELECTOR, "input[type='checkbox']:first-of-type").is_selected()
```
获取元素的标签名  tag_name
实用性较小
```python
# Navigate to url
driver.get("https://www.baidu.com")

# Returns TagName of the element
attr = driver.find_element(By.CSS_SELECTOR, ".s_ipt").tag_name
# 百度首页中class=.s_ipt的元素是一个input
```
用于获取参考元素的尺寸和坐标. element.rect
提取的数据主体包含以下详细信息:
- 元素左上角的X轴位置
- 元素左上角的y轴位置
- 元素的高度
- 元素宽度

```python
# Navigate to url
driver.get("https://www.baidu.com")

# Returns height, width, x and y coordinates referenced element
res = driver.find_element(By.CSS_SELECTOR, ".s_ipt").rect
# {'height': 44, 'width': 550, 'x': 298, 'y': 188.390625}
```
获取元素CSS值
获取当前浏览上下文中元素的特定计算样式属性的值.
```python
# Navigate to Url
driver.get('https://www.baidu.com')

# Retrieves the computed style property 'color' of linktext
cssValue = driver.find_element(By.ID, "su").value_of_css_property('color')
# rgba(255, 255, 255, 1)
```
获取元素文本
获取特定元素渲染后的文本.
```python
# Navigate to Url
driver.get('https://www.baidu.com')
text = driver.find_element(By.LINK_TEXT, "新闻").text
# 新闻
```
元素的attribute和property
```python
# Navigate to Url
driver.get('https://www.baidu.com')
driver.find_element(By.LINK_TEXT, "更多").get_property("href")
# http://www.baidu.com/more/
# <a href="http://www.baidu.com/more/" name="tj_briicon" class="s-bri c-font-normal c-color-t" target="_blank">更多</a>

driver.find_element(By.LINK_TEXT, "更多").get_dom_attribute("href")
# http://www.baidu.com/more/
driver.find_element(By.LINK_TEXT, "更多").get_attribute("href")
# http://www.baidu.com/more/
```
## 元素定位
元素选择策略
在 WebDriver 中有 8 种不同的内置元素定位策略：

定位器 Locator | 	描述
| :- | :- |
| class name |	定位class属性与搜索值匹配的元素（不允许使用复合类名）|
| css selector |	定位 CSS 选择器匹配的元素 |
| id |	定位 id 属性与搜索值匹配的元素 |
| name | 	定位 name 属性与搜索值匹配的元素 |
| link text |	定位link text可视文本与搜索值完全匹配的锚元素 |
| partial link text | 	定位link text可视文本部分与搜索值部分匹配的锚点元素。如果匹配多个元素，则只选择第一个元素。|
| tag name | 	定位标签名称与搜索值匹配的元素 |
| xpath |	定位与 XPath 表达式匹配的元素 |
相对定位器
Selenium 4引入了相对定位器(以前称为友好定位器)。当不容易构造所需元素的定位器，但容易在空间上描述元素相对于具有容易构造定位器的元素的位置时，这些定位器是有帮助的。
它是如何工作的
Selenium使用JavaScript函数getBoundingClientRect()来确定页面上元素的大小和位置，并可以使用该信息来定位相邻的元素。
找到相关元素。
相对定位器方法可以作为原点的参数，可以是先前定位的元素引用，也可以是另一个定位器。在这些例子中，我们将只使用定位器，但你可以用元素对象交换最后一个方法中的定位器，它的工作原理是一样的。
让我们考虑下面的例子来理解相对定位器。
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ac8286f6261745bc6303d318665ad05a.png#pic_center)
above
如果由于某种原因，电子邮件文本字段元素不容易识别，但密码文本字段元素容易识别，那么我们可以使用“输入”元素位于密码元素“上方”的事实来定位文本字段元素。
below
如果由于某种原因，密码文本字段元素不容易识别，而电子邮件文本字段元素容易识别，那么我们可以使用“输入”元素位于电子邮件元素“下方”这一事实来定位文本字段元素。
left of
如果由于某种原因，取消按钮不容易识别，但提交按钮元素容易识别，那么我们可以利用它是提交元素“左边”的一个“按钮”元素来定位取消按钮元素。
right of
如果提交按钮由于某种原因不容易识别，而取消按钮元素容易识别，那么我们可以利用它是“取消”元素右侧的“按钮”元素来定位提交按钮元素。
near
如果相对定位不明显，或者它随窗口大小而变化，则可以使用near方法来标识距离所提供的定位器最多50px的元素。一个很好的例子是，一个表单元素没有一个容易构造的定位器，但是它关联的输入标签元素有。
链式定位
如果需要，您也可以链接定位器。有时元素很容易被识别为一个元素的上面/下面和另一个元素的左右。
```python
email_locator = locate_with(By.TAG_NAME, "input").above({By.ID: "password"})

password_locator = locate_with(By.TAG_NAME, "input").below({By.ID: "email"})

cancel_locator = locate_with(By.TAG_NAME, "button").to_left_of({By.ID: "submit"})

submit_locator = locate_with(By.TAG_NAME, "button").to_right_of({By.ID: "cancel"})

submit_locator = locate_with(By.TAG_NAME, "button").below({By.ID: "email"}).to_right_of({By.ID: "cancel"})
```
## 列表元素的选择
与其他元素相比，选择列表具有特殊的行为.
选择元素可能需要大量样板代码才能自动化. 为了减少这种情况并使您的测试更干净, 在Selenium的support包中有一个 Select 类. 要使用它，您将需要以下导入语句:
```python
from selenium.webdriver.support.select import Select
# 然后，您能够参考 <select> 元素，基于WebElement创建一个Select对象。
select_element = driver.find_element(By.ID,'selectElementID')
select_object = Select(select_element)
```
Select对象现在将为您提供一系列命令，使您可以与 <select> 元素进行交互. 首先，有多种方法可以从 <select> 元素中选择一个选项.
```html
<select>
 <option value=value1>Bread</option>
 <option value=value2 selected>Milk</option>
 <option value=value3>Cheese</option>
</select>
```
有三种方法可以从上述元素中选择第一个选项:
```python
# Select an <option> based upon the <select> element's internal index
select_object.select_by_index(1)

# Select an <option> based upon its value attribute
select_object.select_by_value('value1')

# Select an <option> based upon its text
select_object.select_by_visible_text('Bread')
```
然后，您可以检视所有被选择的选项:
```python
# Return a list[WebElement] of options that have been selected
all_selected_options = select_object.all_selected_options

# Return a WebElement referencing the first selection option found by walking down the DOM
first_selected_option = select_object.first_selected_option
  
```
或者您可能只对 <select> 元素包含哪些 <option> 元素感兴趣:
```python
# Return a list[WebElement] of options that the <select> element contains
all_available_options = select_object.options
```
如果要取消选择任何元素，现在有四个选项:
```python
# Deselect an <option> based upon the <select> element's internal index
select_object.deselect_by_index(1)

# Deselect an <option> based upon its value attribute
select_object.deselect_by_value('value1')

# Deselect an <option> based upon its text
select_object.deselect_by_visible_text('Bread')

# Deselect all selected <option> elements
select_object.deselect_all()
```
最后，一些 <select> 元素允许您选择多个选项. 您可以通过使用以下命令确定您的 <select> 元素是否允许多选:
```python
does_this_allow_multiple_selections = select_object.is_multiple
```
# 等待
WebDriver通常可以说有一个阻塞API。因为它是一个指示浏览器做什么的进程外库，而且web平台本质上是异步的，所以WebDriver不跟踪DOM的实时活动状态。这伴随着一些我们将在这里讨论的挑战。

根据经验，大多数由于使用Selenium和WebDriver而产生的间歇性问题都与浏览器和用户指令之间的 竞争条件 有关。例如，用户指示浏览器导航到一个页面，然后在试图查找元素时得到一个 no such element 的错误。

考虑下面的文档：
```html
<!doctype html>
<meta charset=utf-8>
<title>Race Condition Example</title>

<script>
  var initialised = false;
  window.addEventListener("load", function() {
    var newElement = document.createElement("p");
    newElement.textContent = "Hello from JavaScript!";
    document.body.appendChild(newElement);
    initialised = true;
  });
</script>

```
这个 WebDriver的说明可能看起来很简单:
```python
driver.navigate("file:///race_condition.html")
el = driver.find_element(By.TAG_NAME, "p")
assert el.text == "Hello from JavaScript!"
```
这里的问题是WebDriver中使用的默认页面加载策略页面加载策略听从document.readyState在返回调用 navigate 之前将状态改为"complete" 。因为p元素是在文档完成加载之后添加的，所以这个WebDriver脚本可能是间歇性的。它“可能”间歇性是因为无法做出保证说异步触发这些元素或事件不需要显式等待或阻塞这些事件。

幸运的是，WebElement接口上可用的正常指令集——例如 WebElement.click 和 WebElement.sendKeys—是保证同步的，因为直到命令在浏览器中被完成之前函数调用是不会返回的(或者回调是不会在回调形式的语言中触发的)。高级用户交互APIs,键盘和鼠标是例外的，因为它们被明确地设计为“按我说的做”的异步命令。

等待是在继续下一步之前会执行一个自动化任务来消耗一定的时间。

为了克服浏览器和WebDriver脚本之间的竞争问题，大多数Selenium客户都附带了一个 wait 包。在使用等待时，您使用的是通常所说的显式等待。
### 显示等待
显示等待 是Selenium客户可以使用的命令式过程语言。它们允许您的代码暂停程序执行，或冻结线程，直到满足通过的 条件 。这个条件会以一定的频率一直被调用，直到等待超时。这意味着只要条件返回一个假值，它就会一直尝试和等待

由于显式等待允许您等待条件的发生，所以它们非常适合在浏览器及其DOM和WebDriver脚本之间同步状态。

为了弥补我们之前的错误指令集，我们可以使用等待来让 findElement 调用等待直到脚本中动态添加的元素被添加到DOM中:
```python
from selenium.webdriver.support.ui import WebDriverWait
def document_initialised(driver):
    return driver.execute_script("return initialised")

driver.navigate("file:///race_condition.html")
WebDriverWait(driver).until(document_initialised)
el = driver.find_element(By.TAG_NAME, "p")
assert el.text == "Hello from JavaScript!"
```
我们将 条件 作为函数引用传递， 等待 将会重复运行直到其返回值为true。“truthful”返回值是在当前语言中计算为boolean true的任何值，例如字符串、数字、boolean、对象(包括 WebElement )或填充(非空)的序列或列表。这意味着 空列表 的计算结果为false。当条件为true且阻塞等待终止时，条件的返回值将成为等待的返回值。

有了这些知识，并且因为等待实用程序默认情况下会忽略 no such element 的错误，所以我们可以重构我们的指令使其更简洁:
```python
from selenium.webdriver.support.ui import WebDriverWait

driver.navigate("file:///race_condition.html")
el = WebDriverWait(driver).until(lambda d: d.find_element_by_tag_name("p"))
assert el.text == "Hello from JavaScript!"
  
```
在这个示例中，我们传递了一个匿名函数(但是我们也可以像前面那样显式地定义它，以便重用它)。传递给我们条件的第一个，也是唯一的一个参数始终是对驱动程序对象 WebDriver 的引用。在多线程环境中，您应该小心操作传入条件的驱动程序引用，而不是外部范围中对驱动程序的引用。

因为等待将会吞没在没有找到元素时引发的 no such element 的错误，这个条件会一直重试直到找到元素为止。然后它将获取一个 WebElement 的返回值，并将其传递回我们的脚本。

如果条件失败，例如从未得到条件为真实的返回值，等待将会抛出/引发一个叫 timeout error 的错误/异常。
选项
等待条件可以根据您的需要进行定制。有时候是没有必要等待缺省超时的全部范围，因为没有达到成功条件的代价可能很高。

等待允许你传入一个参数来覆盖超时:
```python
WebDriverWait(driver, timeout=3).until(some_condition)
```
预期的条件
由于必须同步DOM和指令是相当常见的情况，所以大多数客户端还附带一组预定义的 预期条件 。顾名思义，它们是为频繁等待操作预定义的条件。

不同的语言绑定提供的条件各不相同，但这只是其中一些:
- alert is present
- element exists
- element is visible
- title contains
- title is
- element staleness
- visible text

### 隐式等待
还有第二种区别于显示等待 类型的 隐式等待 。通过隐式等待，WebDriver在试图查找_任何_元素时在一定时间内轮询DOM。当网页上的某些元素不是立即可用并且需要一些时间来加载时是很有用的。

默认情况下隐式等待元素出现是禁用的，它需要在单个会话的基础上手动启用。将显式等待和隐式等待混合在一起会导致意想不到的结果，就是说即使元素可用或条件为真也要等待睡眠的最长时间。

警告: 不要混合使用隐式和显式等待。这样做会导致不可预测的等待时间。例如，将隐式等待设置为10秒，将显式等待设置为15秒，可能会导致在20秒后发生超时。

隐式等待是告诉WebDriver如果在查找一个或多个不是立即可用的元素时轮询DOM一段时间。默认设置为0，表示禁用。一旦设置好，隐式等待就被设置为会话的生命周期。
```python
driver = Firefox()
driver.implicitly_wait(10)
driver.get("http://somedomain/url_that_delays_loading")
my_dynamic_element = driver.find_element(By.ID, "myDynamicElement")
  
```

### 流畅等待
流畅等待实例定义了等待条件的最大时间量，以及检查条件的频率。

用户可以配置等待来忽略等待时出现的特定类型的异常，例如在页面上搜索元素时出现的NoSuchElementException.
```python
driver = Firefox()
driver.get("http://somedomain/url_that_delays_loading")
wait = WebDriverWait(driver, 10, poll_frequency=1, ignored_exceptions=[ElementNotVisibleException, ElementNotSelectableException])
element = wait.until(EC.element_to_be_clickable((By.XPATH, "//div")))  
```

# Actions接口
## 键盘Actions
用于与网页交互的任何输入设备的呈现.
Keyboard代表一个键盘事件. Keyboard操作通过使用底层接口允许我们向web浏览器提供虚拟设备输入.
Keys
除了由常规unicode表示的按键, unicode值已指派给其他键盘按键, 以便于Selenium一起使用. 每种语言都有自己的方式来引用这些键; 在此 可以找到完整的列表.[Keyboard Action](https://www.w3.org/TR/webdriver/#keyboard-actions)

Key down
keyDown用于模拟按下辅助按键(CONTROL, SHIFT, ALT)的动作.
```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
driver = webdriver.Chrome()

    # Navigate to url
driver.get("http://www.google.com")

    # Enter "webdriver" text and perform "ENTER" keyboard action
driver.find_element(By.NAME, "q").send_keys("webdriver" + Keys.ENTER)

    # Perform action ctrl + A (modifier CONTROL + Alphabet A) to select the page
webdriver.ActionChains(driver).key_down(Keys.CONTROL).send_keys("a").perform()
  
```
Key up
keyUp用于模拟辅助按键(CONTROL, SHIFT, ALT)弹起或释放的操作.
```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
driver = webdriver.Chrome()

    # Navigate to url
driver.get("http://www.google.com")

    # Store google search box WebElement
search = driver.find_element(By.NAME, "q")

action = webdriver.ActionChains(driver)

    # Enters text "qwerty" with keyDown SHIFT key and after keyUp SHIFT key (QWERTYqwerty)
action.key_down(Keys.SHIFT).send_keys_to_element(search, "qwerty").key_up(Keys.SHIFT).send_keys("qwerty").perform()
```

Send keys
这是Actions API中的一种简便方法, 它将keyDown和keyUp命令组合在一个操作中. 执行此命令与使用元素方法略有不同.

## 鼠标Actions
Mouse表示鼠标事件. 鼠标操作是通过使用底层接口执行的, 其允许我们向Web浏览器提供虚拟化的设备输入操作.
clickAndHold
它将移动到该元素，然后在给定元素的中间单击(不释放).
```python
from selenium import webdriver
driver = webdriver.Chrome()

# Navigate to url
driver.get("http://www.google.com")

# Store 'google search' button web element
searchBtn = driver.find_element(By.LINK_TEXT, "Sign in")

# Perform click-and-hold action on the element
webdriver.ActionChains(driver).click_and_hold(searchBtn).perform() 
```
contextClick
此方法首先将鼠标移动到元素的位置, 然后在给定元素执行上下文点击(右键单击).
```python
from selenium import webdriver
driver = webdriver.Chrome()

# Navigate to url
driver.get("http://www.google.com")

# Store 'google search' button web element
searchBtn = driver.find_element(By.LINK_TEXT, "Sign in")

# Perform context-click action on the element
webdriver.ActionChains(driver).context_click(searchBtn).perform()
```
doubleClick
它将移动到该元素, 并在给定元素的中间双击.
```python
from selenium import webdriver
driver = webdriver.Chrome()

# Navigate to url
driver.get("http://www.google.com")

# Store 'google search' button web element
searchBtn = driver.find_element(By.LINK_TEXT, "Sign in")

# Perform double-click action on the element
webdriver.ActionChains(driver).double_click(searchBtn).perform()
```
moveToElement
此方法将鼠标移到元素的中间. 执行此操作时, 该元素也会滚动到视图中.
```python
from selenium import webdriver
driver = webdriver.Chrome()

# Navigate to url
driver.get("http://www.google.com")

# Store 'google search' button web element
gmailLink = driver.find_element(By.LINK_TEXT, "Gmail")

# Performs mouse move action onto the element
webdriver.ActionChains(driver).move_to_element(gmailLink).perform()
```
moveByOffset:
此方法将鼠标从其当前位置(或0,0)移动给定的偏移量. 如果坐标在视图窗口之外, 则鼠标最终将在浏览器窗口之外.
```python
from selenium import webdriver
driver = webdriver.Chrome()

# Navigate to url
driver.get("http://www.google.com")

# Store 'google search' button web element
gmailLink = driver.find_element(By.LINK_TEXT, "Gmail")
#Set x and y offset positions of element
xOffset = 100
yOffset = 100
# Performs mouse move action onto the element
webdriver.ActionChains(driver).move_by_offset(xOffset,yOffset).perform()
```
dragAndDrop
此方法首先在源元素上单击并按住，然后移动到目标元素的位置后释放鼠标.
```python
from selenium import webdriver
driver = webdriver.Chrome()

# Navigate to url
driver.get("https://crossbrowsertesting.github.io/drag-and-drop")

# Store 'box A' as source element
sourceEle = driver.find_element(By.ID, "draggable")
# Store 'box B' as source element
targetEle  = driver.find_element(By.ID, "droppable")
# Performs drag and drop action of sourceEle onto the targetEle
webdriver.ActionChains(driver).drag_and_drop(sourceEle,targetEle).perform()
```
dragAndDropBy
此方法首先在源元素上单击并按住, 移至给定的偏移量后释放鼠标.
```python
from selenium import webdriver
driver = webdriver.Chrome()

# Navigate to url
driver.get("https://crossbrowsertesting.github.io/drag-and-drop")

# Store 'box A' as source element
sourceEle = driver.find_element(By.ID, "draggable")
# Store 'box B' as source element
targetEle  = driver.find_element(By.ID, "droppable")
targetEleXOffset = targetEle.location.get("x")
targetEleYOffset = targetEle.location.get("y")

# Performs dragAndDropBy onto the target element offset position
webdriver.ActionChains(driver).drag_and_drop_by_offset(sourceEle, targetEleXOffset, targetEleYOffset).perform()
```
release
此操作将释放按下的鼠标左键. 如果WebElement转移了, 它将释放给定WebElement上按下的鼠标左键.
```python
from selenium import webdriver
driver = webdriver.Chrome()

# Navigate to url
driver.get("https://crossbrowsertesting.github.io/drag-and-drop")

# Store 'box A' as source element
sourceEle = driver.find_element(By.ID, "draggable")
# Store 'box B' as source element
targetEle  = driver.find_element(By.ID, "droppable")

# Performs dragAndDropBy onto the target element offset position
webdriver.ActionChains(driver).click_and_hold(sourceEle).move_to_element(targetEle).perform()
#Performs release event
webdriver.ActionChains(driver).release().perform()
```
## 滚轮操作
用于与网页交互的滚轮输入设备的呈现.
滚轮操作将要伴随 Selenium 4.2 发布
当前常用的操作是利用js代码对滚动条进行控制



`声明: 以上内容均来自于selenium官网，本人只做转载和整理`