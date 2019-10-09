
开发模式是如何工作的?

如果您的JavaScript代码非常复杂，在开发和生产环境， 您可能有一种打包和运行不同代码的方法。

在开发和生产环境，打包和运行不同的代码是非常强大的。在开发环境，React包含许多警告，这些警告在导致bug之前可以帮助您找到它们。然而，检测此类错误所需的代码通常会增加包的大小，并导致应用程序运行变慢。

在开发环境运行变慢是可以接受的。事实上，在开发过程中运行代码速度变慢可能是有益的，因为它在一定程度上弥补了高端计算机和普通设备之间的差异。

在生产环境，我们不想任何形式的放缓运行速度。因此，我们在生产中省略了这些检查。这是怎么回事?让我们来看看。

在开发中，运行不同代码正确的方法，取决于JavaScript构建途径(以及是否有)。Facebook是这样的:

if (__DEV__) {
  doSomethingDev();
} else {
  doSomethingProd();
}

这里的__DEV__不是一个真正的变量。当这些模块拼接在一起使浏览器运行时，这个常量就被替换掉了。结果看起来像这样:

// 在开发环境:
if (true) {
  doSomethingDev();
} else {
  doSomethingProd();
}

// 生产环境:
if (false) {
  doSomethingDev();
} else {
  doSomethingProd(); 
}

在生产环境，您需要压缩工具对代码进行压缩（例如：terser）。大多数JavaScript压缩工具将无效的代码做一个有限形式的压缩，例如删除if (false)分支的代码。所以在生产中你只会看到:

// 生产环境压缩之后的代码
doSomethingProd();

(注意：用主流JavaScript工具有效地删除无效代码有很大的限制，但这是另一个主题。)

尽管您可能没有使用__dev__非正式常量，如果您使用流行的JavaScript bundler(如webpack)，那么您可能要遵循其他一些约定。例如，常见的表达方式是这样的:

if (process.env.NODE_ENV !== 'production') {
  doSomethingDev();
} else {
  doSomethingProd();
}

当您使用bundler从npm导入它们时，这正是React和Vue使用的模式（<script>标签构建开发和生产两个不同的版本，.js和.min.js文件）

这个特殊的约定最初来自Node.js。在node.js，有一个全局进程变量，公开系统的环境变量作为process.env对象的一个属性。然而，当您在前端代码库中看到这种模式时，通常不会涉及任何实际的process变量。

相反,整个的process.env.NODE_ENV表达式在构建时被字符串文字所替代,例如我们的非正式__DEV__常量

// 开发环境:
if ('development' !== 'production') { // true
  doSomethingDev();
} else {
  doSomethingProd();
}

// 在生产环境
if ('production' !== 'production') { // false
  doSomethingDev();
} else {
  doSomethingProd();
}

因为整个表达式是常量('production' !== 'production'必然为false)，所以压缩工具也可以删除其他分支代码。

// 生产环境压缩后:
doSomethingProd();

Mischief管理。

注意，这对更复杂的表达式不起作用:
let mode = 'production';
if (mode !== 'production') {
	// 不能保证被消除
}

由于语言的动态特性，JavaScript静态分析工具不是很智能。当他们看到像mode这样的变量而不是像false或'production' !== 'production'这样的静态表达式时，他们通常不会消除。

同样，当使用高级的import语句时，在JavaScript中，跨模块删除无效的代码不能很好的起作用:

// 不保证被消除
import {someFunc} from 'some-module';

if (false) {
  someFunc();
}

因此，您需要以一种非常机械的方式编写代码，使条件绝对是静态的，并确保要消除的所有代码都在其中。

要使所有这些工作正常，您的bundler需要process.env.NODE_ENV替换，并需要知道您希望以哪种模式构建项目。

几年前，常常忘记配置环境。经常会看到一个处于开发模式的项目部署到生产环境中。

这很糟糕，因为这会使网站加载和运行速度变慢。

在过去两年中，情况有了显著的改善。例如，webpack添加了一个简单的模式选项，代替手动配置process.env.NODE_ENV。在开发模式网站React DevTools显示一个红色图标，这使得用户很容易发现甚至报告。
 


像Create React App、Next/Nuxt、Vue CLI、Gatsby和其他一些特有的设置，将开发构建和生产构建分离成两个单独的命令使它变得更糟糕(例如，npm start和npm run build)。通常，只部署生产版本，因此开发人员不能再犯这个错误。

总是有这样一种说法，即生产模式是需要默认的，而开发模式需要opt-in。就我个人而言，我不认为这个论点有说服力。从开发模式警告中获益最多的人通常是库的新手。他们不知道如何打开它，这些警告没有及时被察觉，导致未在早期发现很多bug。

是的，性能问题很糟糕。但向最终用户提供有缺陷的体验也是如此。例如，
React key warning有助于防止错误，比如向错误的人或购买错误的产品的人发送消息。禁用此警告进行开发对您和您的用户都是一个重大风险。如果默认情况下它是关闭的，那么当您找到toggle并打开它时，您将有太多警告需要清除。所以大多数人会把它切换回去。这就是为什么需要从一开始就打开它，而不是稍后才启用它。

最后，即使开发警告是可选的，并且开发人员知道在开发的早期就启用它们，我们回到最初的问题。部署到生产环境时，有人会不小心让它们处于打开状态!

就我个人而言，我相信工具显示和使用正确的模式，取决于您是在调试还是部署。几十年来，除了web浏览器之外，几乎所有其他环境(无论是移动环境、桌面环境还是服务器环境)就有一种方法来加载和区分开发和生产构建。

提出并依赖于特别的约定，而不是库。也许JavaScript环境是时候把这种区别看作是头等需要了。

理论说得够多了!
让我们再来看看这段代码:
if (process.env.NODE_ENV !== 'production') {
  doSomethingDev();
} else {
  doSomethingProd();
}
您可能会想:如果前端没有真正的process对象，为什么像React和Vue这样的库在npm构建中要依赖它?
(再次澄清一下,script标签可以在浏览器中加载，React和Vue都提供，不要依赖于它，相反，您必须在开发.js文件和生产.min.js文件之间做出选择，下面的小节只讨论如何通过从npm导入一个bundler来使用React或Vue)


就像编程中的许多事情一样，这种特殊的约定主要有历史原因。我们仍然在使用它，因为现在它被不同的工具广泛采用。转换到其他东西代价是很高的。

那么它背后的历史是什么呢?

在import和exports语法标准化之前的很多年，有几种相互竞争的方式来表达模块之间的关系。Node.js推广了require()和module.exports，被称为CommonJS。

早期发布在npm上的代码是为Node.js编写的。Express是最流行的Node服务器端框架（也许现在仍然如此），并使用NODE_ENV环境变量启用生产模式。其他一些npm包也采用了相同的约定。

早期的JavaScript打包，如browserify，希望能够在前端项目中使用来自npm的代码。(你可能无法想象当时在前端几乎没有人使用npm!)因此，他们将Node.js生态系统中已经存在的相同约定扩展到前端代码。

最初的“envify”版本于2013年发布。在那个时代，React当时是开源的。使用browserify的npm似乎是打包前端CommonJS代码的最佳解决方案。

从一开始，React开始提供npm构建。React越来越流行，与CommonJS模块和前端代码通过npm模块化编写JavaScript的实践也是如此

在生产模式，React需要删除development-only的代码。Browserify已经为这个问题提供了一个解决方案，为了npm 构建，React也采用了process.env.NODE_ENV的约定。随着时间的推移，许多其他工具和库，包括webpack和Vue，也是这么做的。

到2019年，browserify已经失去了相当多的市场份额，在构建步骤中用development和production替换process.env.NODE_ENV是一个与以前一样流行的约定。


（如何采用ES模块作为一种分发模式而不是authoring 模式这将很有趣，改变了这个方式，在Twitter上告诉我?）
还有一件事可能会让您感到困惑，那就是在GitHub上的React源代码中，您将看到使用了__DEV__作为一个神奇的变量。但是在npm上的React代码中，它使用process.env.NODE_ENV。这是怎么回事?

历史上，在源代码中我们使用__DEV__来匹配Facebook的源代码。长期以来，React被直接复制到Facebook的代码库中，所以需要遵循同样的规则。对于npm，在之前的版本中，我们有一个构建步骤，用process.env.NODE_ENV !== 'production'替换了__DEV__检查。

这有时是个问题。有时候，依赖于Node.js约定的代码模式在npm上运行得很好，在 Facebook上就报错，反之亦然。

从React 16开始，我们改变了方法。相反，我们现在为每个环境编译一个包(包括<script>标签，npm,和Facebook内部代码库)，因此，即使是用于npm的CommonJS代码也会被编译成独立的开发和生产包

这意味着，当React源代码说if (__DEV__)，实际上，我们每生产一个包要生产两个包。其中一个已经用__DEV__ = true预编译，另一个用__DEV__ = false预编译。npm上每个包的入口“决定”导出哪个包。

例如：

if (process.env.NODE_ENV === 'production') {
  module.exports = require('./cjs/react.production.min.js');
} else {
  module.exports = require('./cjs/react.development.js');
}

这是唯一的地方，你的bundler将插入development或production作为一个字符串，你的压缩工具将从development-only require中去除。

react.production.min.js和react.development.js都没有任何process.env.NODE_ENV检查。这很好，因为当实际运行在Node上时。Process.env有点慢。提前在这两种模式下编译bundle还可以让我们更一致地优化文件大小，而不管您使用的是哪种bundler或minifier。

这就是它的工作原理!

我希望有一种更一流的方法可以做到这一点，而不依赖于惯例，但我们现在就在这里。如果模式在所有JavaScript环境中都是一个一流的概念，如果浏览器能够以某种方式在开发模式下运行一些代码，而开发模式本不应该运行这些代码，那就太好了。




另一方面，一个项目中的约定如何能够在整个生态系统中传播，这是非常有趣的。EXPRESS_ENV于2010年成为NODE_ENV，并于2013年扩展到前端。这个解决方案也许并不完美，但是对于每个项目来说，采用它的成本要低于说服其他人做不同的事情的成本。这两种方式的对比，给我们上了宝贵的一课，理解这种动态是如何进行的，可以将成功的标准化尝试与失败区分开来。

开发模式与生产模式的分离是一项非常有用的技术。我建议在您的库和应用程序代码中使用它，用于那些在生产环境中执行开销太大，但在开发中执行却很有价值的检查。

对于任何强大的特性，都有一些方法可以误用它。这将是我下一篇文章的主题!




