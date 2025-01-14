




# TODO 19 调试 Lisp 程序

有几种方法可以查找和调查 Emacs Lisp 程序中的问题。

如果在运行程序时出现问题，您可以使用内置的 Emacs Lisp 调试器来暂停 Lisp 求值器，并检查和/或更改其内部状态。
您可以使用 Edebug，它是 Emacs Lisp 的源代码级调试器。
您可以使用 trace.el 包提供的跟踪工具来跟踪问题中涉及的函数的执行。  该包提供了用于跟踪函数调用的函数 trace-function-foreground 和 trace-function-background ，以及用于将选择变量的值添加到跟踪中的 trace-values 。  有关详细信息，请参阅 trace.el 中这些工具的文档。
如果语法问题甚至阻止 Lisp 读取程序，您可以使用 Lisp 编辑命令来定位它。
您可以查看字节编译器在编译程序时产生的错误和警告消息。  请参阅编译器错误。
您可以使用 Testcover 包对程序执行覆盖测试。
您可以使用 ERT 包为程序编写回归测试。  请参阅 ERT 中的 ERT 手册：Emacs Lisp 回归测试。
您可以分析程序以获取有关如何提高效率的提示。

调试输入和输出问题的其他有用工具是 dribble 文件（参见终端输入）和 open-termscript 函数（参见终端输出）。



## TODO 19.1 Lisp 调试器

普通的 Lisp 调试器提供暂停对表单求值的能力。  当评估暂停时（通常称为中断的状态），您可以检查运行时堆栈，检查局部或全局变量的值，或更改这些值。  由于 break 是递归编辑，因此 Emacs 的所有常用编辑工具都可用；  您甚至可以运行将递归地进入调试器的程序。  请参阅递归编辑。



### TODO 19.1.1 出错时进入调试器

进入调试器最重要的时间是发生 Lisp 错误的时候。  这使您可以调查错误的直接原因。

但是，进入调试器并不是错误的正常结果。  许多命令在调用不当时会发出 Lisp 错误信号，并且在普通编辑期间，每次发生这种情况时进入调试器会非常不方便。  因此，如果您希望错误进入调试器，请将变量 debug-on-error 设置为非零。  （toggle-debug-on-error 命令提供了一种简单的方法来执行此操作。）

    User Option: debug-on-error ¶

此变量确定在发出错误信号但未处理时是否调用调试器。  如果 debug-on-error 为 t，则所有类型的错误都会调用调试器，除了 debug-ignored-errors 中列出的错误（见下文）。  如果为 nil，则没有人调用调试器。

该值也可以是错误条件列表（请参阅如何发出错误信号）。  然后仅针对此列表中的错误条件调用调试器（除了那些也在 debug-ignored-errors 中列出的情况）。  例如，如果您将 debug-on-error 设置为列表 (void-variable)，则调试器只会针对没有值的变量的错误调用。

请注意， eval-expression-debug-on-error 在某些情况下会覆盖此变量；  见下文。

当这个变量不为 nil 时，Emacs 不会围绕进程过滤函数和哨兵创建错误处理程序。  因此，这些函数中的错误也会调用调试器。  请参阅进程。

    User Option: debug-ignored-errors ¶

此变量指定不应进入调试器的错误，无论 debug-on-error 的值如何。  它的值是错误条件符号和/或正则表达式的列表。  如果错误具有这些条件符号中的任何一个，或者如果错误消息与任何正则表达式匹配，则该错误不会进入调试器。

此变量的正常值包括用户错误，以及在编辑过程中经常发生但很少由 Lisp 程序中的错误引起的几个错误。  然而，“很少”不是“从不”；  如果您的程序因与此列表匹配的错误而失败，您可以尝试更改此列表以调试错误。  最简单的方法通常是将 debug-ignored-errors 设置为 nil。

    User Option: eval-expression-debug-on-error ¶

如果此变量具有非 nil 值（默认值），则运行命令 eval-expression 会导致 debug-on-error 临时绑定到 t。  请参阅 GNU Emacs 手册中的 Emacs Lisp 表达式求值。

如果 eval-expression-debug-on-error 为 nil，则在 eval-expression 期间不会更改 debug-on-error 的值。

    User Option: debug-on-signal ¶

通常，条件案例捕获的错误永远不会调用调试器。  在调试器有机会之前，条件案例有机会处理错误。

如果您将 debug-on-signal 更改为非 nil 值，则无论是否存在条件情况，调试器都会在每个错误中获得第一次机会。  （要调用调试器，错误仍必须满足 debug-on-error 和 debug-ignored-errors 指定的条件。）

例如，设置此变量对于从 emacsclient 的 &#x2013;eval 选项评估的代码中获取回溯很有用。  如果由 emacsclient 评估的 Lisp 代码在此变量为非 nil 时发出错误信号，则回溯将在运行的 Emacs 中弹出。

警告：将此变量设置为非零可能会产生烦人的效果。  Emacs 的各个部分在正常的事务过程中捕获错误，您甚至可能没有意识到错误发生在那里。  如果您需要调试包含在条件案例中的代码，请考虑使用条件案例除非调试（请参阅编写代码以处理错误）。

    User Option: debug-on-event ¶

如果将 debug-on-event 设置为特殊事件（请参阅特殊事件），Emacs 将在收到此事件后立即尝试进入调试器，绕过特殊事件映射。  目前，唯一支持的值对应于信号 SIGUSR1 和 SIGUSR2（这是默认值）。  当设置了禁止退出并且 Emacs 没有以其他方式响应时，这会很有帮助。

    Variable: debug-on-message ¶

如果将 debug-on-message 设置为正则表达式，则 Emacs 如果在回显区域显示匹配的消息，将进入调试器。  例如，这在尝试查找特定消息的原因时很有用。

要调试在加载初始化文件期间发生的错误，请使用选项“&#x2013;debug-init”。  这会在加载 init 文件时将 debug-on-error 绑定到 t，并绕过通常在 init 文件中捕获错误的条件情况。



### TODO 19.1.2 调试无限循环

当程序无限循环并且无法返回时，您的第一个问题就是停止循环。  在大多数操作系统上，您可以使用 Cg 执行此操作，这会导致退出。  请参阅退出。

普通退出不会提供有关程序为何循环的信息。  要获取更多信息，您可以将变量 debug-on-quit 设置为非零。  一旦调试器在无限循环的中间运行，您就可以使用步进命令从调试器继续。  如果您逐步完成整个循环，您可能会获得足够的信息来解决问题。

用 Cg 退出不被认为是错误，debug-on-error 对 Cg 的处理没有影响。  同样，debug-on-quit 对错误没有影响。

    User Option: debug-on-quit ¶

此变量确定在发出退出信号但未处理时是否调用调试器。  如果 debug-on-quit 不为 nil，则在您退出时调用调试器（即，键入 Cg）。  如果 debug-on-quit 为 nil（默认值），则退出时不会调用调试器。



### TODO 19.1.3 在函数调用中进入调试器

要调查程序中间发生的问题，一种有用的技术是在调用某个函数时进入调试器。  您可以对发生问题的函数执行此操作，然后单步执行该函数，或者您可以对在问题发生前不久调用的函数执行此操作，快速跳过对该函数的调用，然后单步执行其调用者。

    Command: debug-on-entry function-name ¶

该函数每次调用时都请求函数名来调用调试器。

任何定义为 Lisp 代码的函数或宏都可以设置为在入口处中断，无论它是解释代码还是编译代码。  如果函数是命令，当从 Lisp 调用和交互调用时（在读取参数之后），它将进入调试器。  您也可以通过这种方式为原始函数（即用 C 编写的函数）设置 debug-on-entry，但它仅在从 Lisp 代码调用原始函数时生效。  特殊形式不允许进入调试。

当以交互方式调用 debug-on-entry 时，它会提示输入 minibuffer 中的函数名。  如果该函数已设置为在进入时调用调试器，则 d​​ebug-on-entry 什么也不做。  debug-on-entry 总是返回函数名。

    下面是一个例子来说明这个函数的使用：
    #+begin<sub>src</sub> emacs-lisp
(defun fact (n)
  (if (zerop n) 1
      (\* n (fact (1- n)))))
     ⇒ fact

(debug-on-entry 'fact)
     ⇒ fact

(fact 3)

-&#x2013;&#x2014; Buffer: **Backtrace** -&#x2013;&#x2014;
Debugger entered&#x2013;entering a function:



# fact(3)

  eval((fact 3))
  eval-last-sexp-1(nil)
  eval-last-sexp(nil)
  call-interactively(eval-last-sexp)
-&#x2013;&#x2014; Buffer: **Backtrace** -&#x2013;&#x2014;
    #+end<sub>src</sub>

    Command: cancel-debug-on-entry &optional function-name ¶

此函数撤消 debug-on-entry 对函数名的影响。  当以交互方式调用时，它会提示输入 minibuffer 中的函数名。  如果 function-name 被省略或 nil，它将取消所有函数的 break-on-entry。  调用 cancel-debug-on-entry 对当前未设置为在进入时中断的函数没有任何作用。



### TODO 19.1.4 修改变量时进入调试器

有时，函数的问题是由于变量设置错误造成的。  将调试器设置为在变量更改时触发是一种快速查找设置来源的方法。

    Command: debug-on-variable-change variable ¶

此函数安排在修改变量时调用调试器。

它是使用watchpoint机制实现的，因此继承了相同的特点和局限性：变量的所有别名都将被一起监视，只能监视动态变量，并且不会检测到变量引用的对象的变化。  有关详细信息，请参阅在变量更改时运行函数。。

    Command: cancel-debug-on-variable-change &optional variable ¶

此函数撤消 debug-on-variable-change 对变量的影响。  当以交互方式调用时，它会提示输入 minibuffer 中的变量。  如果变量被省略或为零，它将取消所有变量的更改中断。  调用 cancel-debug-on-variable-change 对当前未设置为在更改时中断的变量没有任何作用。



### TODO 19.1.5 显式进入调试器

您可以通过在该点编写表达式 (debug) 来使调试器在程序中的某个点被调用。  为此，请访问源文件，在适当的位置插入文本“(debug)”，然后键入 CMx（eval-defun，一种 Lisp 模式键绑定）。  警告：如果您这样做是出于临时调试目的，请务必在保存文件之前撤消此插入！

插入“（调试）”的位置必须是可以评估附加表单并忽略其值的位置。  （如果 (debug) 的值没有被忽略，它将改​​变程序的执行！）最常见的合适位置是在 progn 或隐式 progn 内（参见 Sequencing）。

如果您不知道要在源代码中的确切位置放置调试语句，但希望在显示特定消息时显示回溯，则可以将 debug-on-message 设置为匹配所需消息的正则表达式.



### TODO 19.1.6 使用调试器

进入调试器后，它会在一个窗口中显示先前选择的缓冲区，并在另一个窗口中显示一个名为 **Backtrace** 的缓冲区。  回溯缓冲区包含当前正在进行的每一级 Lisp 函数执行的一行。  在这个缓冲区的开头是一条消息，描述了调试器被调用的原因（例如错误消息和相关数据，如果它是由于错误而被调用的）。

回溯缓冲区是只读的，并使用一种特殊的主要模式，调试器模式，其中字母被定义为调试器命令。  可以使用常用的 Emacs 编辑命令；  因此，您可以切换窗口以检查发生错误时正在编辑的缓冲区、切换缓冲区、访问文件或进行任何其他类型的编辑。  但是，调试器是递归编辑级别（请参阅递归编辑），当您完成调试器时，最好返回回溯缓冲区并退出调试器（使用 q 命令）。  退出调试器退出递归编辑并掩埋回溯缓冲区。  （您可以通过设置变量 debugger-bury-or-kill 来自定义 q 命令对回溯缓冲区的作用。例如，如果您更喜欢杀死缓冲区而不是埋葬它，请将其设置为 kill。有关更多信息，请参阅变量的文档可能性。）

进入调试器后，根据 eval-expression-debug-on-error 临时设置 debug-on-error 变量。  如果后一个变量不为 nil，则 debug-on-error 将临时设置为 t。  这意味着在进行调试会话时发生的任何进一步错误将（默认情况下）触发另一个回溯。  如果这不是您想要的，您可以将 eval-expression-debug-on-error 设置为 nil，或者在 debugger-mode-hook 中将 debug-on-error 设置为 nil。

调试器本身必须运行字节编译，因为它对 Lisp 解释器的状态做出假设。  如果调试器正在解释运行，则这些假设是错误的。



### TODO 19.1.7 回溯

Debugger 模式源自 Backtrace 模式，Edebug 和 ERT 也用于显示回溯。  （请参阅 Edebug 和 ERT 中的 ERT 手册：Emacs Lisp 回归测试。）

回溯缓冲区显示正在执行的函数及其参数值。  创建回溯缓冲区时，它会将每个堆栈帧显示在一个可能很长的行上。  （堆栈帧是 Lisp 解释器记录有关函数的特定调用的信息的地方。）最近调用的函数将位于顶部。

在回溯中，您可以通过将点移动到描述该帧的行来指定堆栈帧。  线点打开的帧被认为是当前帧。

如果函数名带有下划线，则表示 Emacs 知道其源代码的位置。  您可以用鼠标单击该名称，或移至该名称并键入 RET，以访问源代码。  您还可以在 point 位于没有下划线的函数或变量的任何名称上时键入 RET，以查看帮助缓冲区中该符号的帮助信息（如果存在）。  绑定到 M-. 的 xref-find-definitions 命令也可用于回溯中的任何标识符（请参阅 GNU Emacs 手册中的查找标识符）。

在回溯中，长列表的尾部和长字符串、向量或结构的末尾，以及深度嵌套的对象，将打印为带下划线的“&#x2026;”。  您可以用鼠标单击“&#x2026;”，或在点位于其上时键入 RET，以显示隐藏的对象部分。  要控制完成多少缩写，请自定义 backtrace-line-length。

以下是用于导航和查看回溯的命令列表：

    v

切换当前堆栈帧的局部变量的显示。

    p

移动到帧的开头，或上一帧的开头。

    n

移动到下一帧的开头。

    +

在顶层 Lisp 表单中添加换行符和缩进，使其更具可读性。

    -

将点处的顶级 Lisp 表单折叠回单行。

    #

在点处切换框架的打印圆圈。

    :

在该点切换帧的 print-gensym。

    .

展开框架中所有缩写为“&#x2026;”的表格。



### TODO 19.1.8 调试器命令

除了通常的 Emacs 命令和上一节中描述的 Backtrace 模式命令之外，调试器缓冲区（在 Debugger 模式下）还提供特殊命令。  调试器命令最重要的用途是单步执行代码，这样您就可以看到控制是如何流动的。  调试器可以单步执行解释函数的控制结构，但不能在字节编译函数中这样做。  如果您想单步执行字节编译的函数，请将其替换为同一函数的解释定义。  （为此，请访问函数的源代码并在其定义中键入 CMx。）您不能使用 Lisp 调试器单步执行原始函数。

一些调试器命令在当前帧上运行。  如果一个框架以星号开头，这意味着退出该框架将再次调用调试器。  这对于检查函数的返回值很有用。

以下是调试器模式命令的列表：

    c

退出调试器并继续执行。  这将恢复程序的执行，就好像从未进入调试器一样（除了您在调试器内部更改变量值或数据结构引起的任何副作用）。

    d

继续执行，但在下次调用任何 Lisp 函数时进入调试器。  这允许您单步执行表达式的子表达式，查看子表达式计算的值以及它们还做了什么。

以这种方式进入调试器的函数调用的堆栈帧将被自动标记，以便在退出帧时再次调用调试器。  您可以使用 u 命令取消此标志。

    b

标记当前帧，以便在退出该帧时进入调试器。  以这种方式标记的帧在回溯缓冲区中用星号标记。

    u

退出当前帧时不要进入调试器。  这会取消该帧上的 ab 命令。  可见效果是从回溯缓冲区中的行中删除星号。

    j

像 b 一样标记当前帧。  然后像 c 一样继续执行，但暂时禁用所有由 debug-on-entry 设置的函数的break-on-entry。

    e

读取 minibuffer 中的 Lisp 表达式，评估它（使用相关的词法环境，如果适用），并在 echo 区域打印值。  调试器会更改某些重要变量和当前缓冲区，作为其操作的一部分；  e 临时从调试器外部恢复它们的值，因此您可以检查和更改它们。  这使调试器更加透明。  相比之下， M-: 在调试器中没有什么特别之处；  它向您显示调试器中的变量值。

    R

与 e 一样，也将评估结果保存在缓冲区 **Debugger-record** 中。

    q

终止正在调试的程序；  返回顶层 Emacs 命令执行。

如果由于 Cg 而进入调试器，但您真的想退出而不是调试，请使用 q 命令。

    r

从调试器返回一个值。  该值是通过读取带有微型缓冲区的表达式并对其进行评估来计算的。

当调试器由于退出 Lisp 调用框架而被调用时，r 命令很有用（根据 b 请求或通过 d 进入框架）；  然后将 r 命令中指定的值用作该帧的值。  如果您调用 debug 并使用它的返回值，它也很有用。  否则，r 和 c 效果一样，指定的返回值无关紧要。

由于错误而进入调试器时，您不能使用 r。

    l

显示调用时将调用调试器的函数列表。  这是一个通过 debug-on-entry 设置为在入口时中断的函数列表。



### TODO 19.1.9 调用调试器

在这里，我们将详细描述用于调用调试器的函数 debug。

    Command: debug &rest debugger-args ¶

该函数进入调试器。  它将缓冲区切换到名为 **Backtrace** 的缓冲区（或 \*Backtrace\*<2>，如果它是调试器的第二个递归条目，等等），并用有关 Lisp 函数调用堆栈的信息填充它。  然后它进入递归编辑，在调试器模式下显示回溯缓冲区。

Debugger模式c、d、j、r命令退出递归编辑；  然后调试切换回前一个缓冲区并返回到任何称为调试的地方。  这是函数调试可以返回给它的调用者的唯一方式。

debugger-args 的用途是 debug 将其余参数显示在 **Backtrace** 缓冲区的顶部，以便用户可以看到它们。  除下文所述外，这是使用这些参数的唯一方式。

但是，要调试的第一个参数的某些值具有特殊意义。  （通常，这些值仅由 Emacs 内部使用，而不是由调用 debug 的程序员使用。）下面是这些特殊值的表：

    lambda ¶

lambda 的第一个参数表示当 debug-on-next-call 为非 nil 时，由于进入函数而调用了调试。  调试器在缓冲区顶部将“已输入调试器&#x2013;输入函数：”显示为一行文本。

    debug

debug 作为第一个参数意味着调试被调用是因为进入了一个设置为在进入时调试的函数。  调试器显示字符串“调试器输入-输入函数：”，就像在 lambda 情况下一样。  它还标记该函数的堆栈帧，以便在退出时调用调试器。

    t

当第一个参数为 t 时，这表示由于在 debug-on-next-call 为非 nil 时评估函数调用形式而调用 debug。  调试器在缓冲区的第一行显示“已进入调试器——开始评估函数调用形式：”。

    exit

当第一个参数为 exit 时，它表示先前标记为在退出时调用调试器的堆栈帧的退出。  在这种情况下，给 debug 的第二个参数是从帧返回的值。  调试器在缓冲区的第一行显示“调试器输入&#x2013;返回值：”，然后是返回的值。

    error

当第一个参数是错误时，调试器通过显示“已输入的调试器&#x2013;Lisp 错误：”后跟发出的错误信号和任何要发出信号的参数来指示它正在进入，因为已发出错误或退出信号但未处理。  例如，

    (let ((debug-on-error t))
      (/ 1 0))
    
    
    ------ Buffer: *Backtrace* ------
    Debugger entered--Lisp error: (arith-error)
      /(1 0)
    ...
    ------ Buffer: *Backtrace* ------

如果发出错误信号，则变量 debug-on-error 可能不为零。  如果发出了退出信号，则可能变量 debug-on-quit 为非零。

    nil

当你想显式地进入调试器时，使用 nil 作为调试器参数的第一个。  其余的调试器参数打印在缓冲区的顶行。  您可以使用此功能来显示消息——例如，提醒自己在哪些条件下调用了调试。



### TODO 19.1.10 调试器的内部结构

本节介绍调试器内部使用的函数和变量。

    Variable: debugger ¶

这个变量的值是调用调试器的函数。  它的值必须是任意数量的参数的函数，或者更典型的是函数的名称。  这个函数应该调用某种调试器。  变量的默认值为调试。

Lisp 传递给函数的第一个参数表明了它被调用的原因。  参数的约定在调试的描述中有详细说明（请参阅调用调试器）。

    Function: backtrace ¶

此函数打印当前活动的 Lisp 函数调用的跟踪。  跟踪与调试将在 **Backtrace** 缓冲区中显示的跟踪相同。  返回值始终为零。

在以下示例中，Lisp 表达式显式调用回溯。  这会将回溯打印到流标准输出，在这种情况下，它是缓冲区“回溯输出”。

回溯的每一行代表一个函数调用。  该行显示了函数，然后是函数参数值的列表（如果它们都是已知的）；  如果它们仍在计算中，则该行由一个包含函数及其未评估参数的列表组成。  长列表或深度嵌套的结构可能会被省略。

    (with-output-to-temp-buffer "backtrace-output"
      (let ((var 1))
        (save-excursion
          (setq var (eval '(progn
    			 (1+ var)
    			 (list 'testing (backtrace))))))))
    
         ⇒ (testing nil)
    
    
    ----------- Buffer: backtrace-output ------------
      backtrace()
      (list 'testing (backtrace))
    
      (progn ...)
      eval((progn (1+ var) (list 'testing (backtrace))))
      (setq ...)
      (save-excursion ...)
      (let ...)
      (with-output-to-temp-buffer ...)
      eval((with-output-to-temp-buffer ...))
      eval-last-sexp-1(nil)
    
      eval-last-sexp(nil)
      call-interactively(eval-last-sexp)
    ----------- Buffer: backtrace-output ------------

    User Option: debugger-stack-frame-as-list ¶

如果此变量不为零，则回溯的每个堆栈帧都显示为列表。  这旨在以特殊形式不再与常规函数调用在视觉上有所不同为代价来提高回溯的可读性。

使用 debugger-stack-frame-as-list 非 nil 时，上面的示例如下所示：

    ----------- Buffer: backtrace-output ------------
      (backtrace)
      (list 'testing (backtrace))
    
      (progn ...)
      (eval (progn (1+ var) (list 'testing (backtrace))))
      (setq ...)
      (save-excursion ...)
      (let ...)
      (with-output-to-temp-buffer ...)
      (eval (with-output-to-temp-buffer ...))
      (eval-last-sexp-1 nil)
    
      (eval-last-sexp nil)
      (call-interactively eval-last-sexp)
    ----------- Buffer: backtrace-output ------------

    Variable: debug-on-next-call ¶

如果这个变量不为零，它表示在下一次 eval、apply 或 funcall 之前调用调试器。  进入调试器会将 debug-on-next-call 设置为 nil。

调试器中的 d 命令通过设置此变量来工作。

    Function: backtrace-debug level flag ¶

此函数将堆栈帧级别的 debug-on-exit 标志设置为堆栈的下一级，并为其赋予 value 标志。  如果 flag 不为零，这将导致在该帧稍后退出时进入调试器。  即使是通过该帧的非本地退出也会进入调试器。

此函数仅供调试器使用。

    Variable: command-debug-status ¶

该变量记录当前交互命令的调试状态。  每次以交互方式调用命令时，此变量都绑定为 nil。  调试器可以设置此变量，以便在同一命令调用期间为将来的调试器调用留下信息。

使用这个变量而不是普通的全局变量的优点是数据永远不会转移到后续的命令调用中。

此变量已过时，将在未来版本中删除。

    Function: backtrace-frame frame-number &optional base ¶

函数 backtrace-frame 旨在用于 Lisp 调试器。  它返回有关在堆栈帧帧号级别向下发生的计算的信息。

如果该框架尚未评估参数，或者是特殊形式，则值为 (nil function arg-forms&#x2026;)。

如果该框架已评估其参数并已调用其函数，则返回值为 (t function arg-values&#x2026;)。

在返回值中，function 是作为评估列表的 CAR 提供的任何内容，或者在宏调用的情况下是 lambda 表达式。  如果函数具有 &rest 参数，则表示为列表 arg-values 的尾部。

如果指定了基数，则帧数相对于函数为基数的最顶层帧计数。

如果 frame-number 超出范围，则 backtrace-frame 返回 nil。

    Function: mapbacktrace function &optional base ¶

函数 mapbacktrace 为回溯中的每一帧调用一次函数，从函数为 base 的第一帧开始（如果 base 省略或为零，则从顶部开始）。

使用四个参数调用函数：evald、func、args 和 flags。

如果一个框架还没有评估它的参数或者是一个特殊的形式，那么 evald 是 nil 并且 args 是一个形式的列表。

如果一个框架已经评估了它的参数并调用了它的函数，那么 evald 是 t 并且 args 是一个值列表。  flags 是当前帧的属性列表：目前，唯一支持的属性是 :debug-on-exit，如果设置了堆栈帧的 debug-on-exit 标志，则为 t。



## TODO 19.2 调试

Edebug 是 Emacs Lisp 程序的源代码级调试器，您可以使用它：

1.  逐步执行评估，在每个表达式之前和之后停止。
2.  设置条件断点或无条件断点。
3.  当指定条件为真（全局中断事件）时停止。
4.  慢速或快速跟踪，在每个停止点或每个断点处短暂停止。
5.  显示表达式结果并评估表达式，就像在 Edebug 之外一样。
6.  每次 Edebug 更新显示时，自动重新评估表达式列表并显示其结果。
7.  输出有关函数调用和返回的跟踪信息。
8.  发生错误时停止。
9.  显示回溯，省略 Edebug 自己的帧。
10. 为宏和定义表单指定参数评估。
11. 获得基本的覆盖测试和频率计数。

下面的前三个部分应该告诉您足够多的有关 Edebug 的信息，以便开始使用它。



### TODO 19.2.1 使用 Edebug

要使用 Edebug 调试 Lisp 程序，您必须首先检测要调试的 Lisp 代码。  一个简单的方法是首先将点移动到函数或宏的定义中，然后执行 Cu CMx（带有前缀参数的 eval-defun）。  请参阅 Instrumenting for Edebug，了解检测代码的替代方法。

一旦检测到函数，对该函数的任何调用都会激活 Edebug。  根据您选择的 Edebug 执行模式，激活 Edebug 可能会停止执行并让您逐步执行该功能，或者它可能会更新显示并在检查调试命令时继续执行。  默认执行模式是 step，它会停止执行。  请参阅 Edebug 执行模式。

在 Edebug 中，您通常会查看一个 Emacs 缓冲区，其中显示了您正在调试的 Lisp 代码的源代码。  这称为源代码缓冲区，它是临时只读的。

左边缘的箭头表示函数正在执行的行。  Point 最初显示函数在行内执行的位置，但如果您自己移动 point，这将不再适用。

如果您检测 fac 的定义（如下所示）然后执行（fac 3），这就是您通常会看到的内容。  点位于 if 之前的左括号处。

    (defun fac (n)
    =>∗(if (< 0 n)
          (* n (fac (1- n)))
        1))

函数中 Edebug 可以停止执行的位置称为停止点。  这些出现在每个作为列表的子表达式之前和之后，也出现在每个变量引用之后。  这里我们使用句点来显示函数 fac 中的停止点：

    (defun fac (n)
      .(if .(< 0 n.).
          .(* n. .(fac .(1- n.).).).
        1).)

除了 Emacs Lisp 模式的命令外，源代码缓冲区中还有 Edebug 的特殊命令。  例如，您可以键入 Edebug 命令 SPC 执行直到下一个停止点。  如果您在进入 fac 后键入 SPC 一次，您将看到以下显示：

    (defun fac (n)
    =>(if ∗(< 0 n)
          (* n (fac (1- n)))
        1))

当 Edebug 在表达式后停止执行时，它会在回显区域显示表达式的值。

其他常用的命令是 b 在停止点设置断点， g 执行直到到达断点， q 退出 Edebug 并返回到顶层命令循环。  类型 ？  显示所有 Edebug 命令的列表。



### TODO 19.2.2 为 Edebug 检测

为了使用 Edebug 调试 Lisp 代码，您必须首先检测代码。  检测代码会在其中插入额外的代码，以便在适当的位置调用 Edebug。

当您在函数定义上调用带有前缀参数的命令 CMx (eval-defun) 时，它会在对定义进行评估之前对其进行检测。  （这不会修改源代码本身。）如果变量 edebug-all-defs 不为 nil，则会反转前缀参数的含义：在这种情况下，CMx 检测定义，除非它具有前缀参数。  edebug-all-defs 的默认值为 nil。  命令 Mx edebug-all-defs 切换变量 edebug-all-defs 的值。

如果 edebug-all-defs 不是 nil，那么命令 eval-region、eval-current-buffer 和 eval-buffer 也会检测它们评估的任何定义。  同样， edebug-all-forms 控制 eval-region 是否应该检测任何形式，甚至是非定义形式。  这不适用于 minibuffer 中的加载或评估。  命令 Mx edebug-all-forms 切换此选项。

另一个命令 Mx edebug-eval-top-level-form 可用于检测任何顶级表单，而不管 edebug-all-defs 和 edebug-all-forms 的值如何。  edebug-defun 是 edebug-eval-top-level-form 的别名。

当 Edebug 处于活动状态时，命令 I (edebug-instrument-callee) 会在点之后检测由列表形式调用的函数或宏的定义，如果它尚未检测的话。  只有当 Edebug 知道在哪里可以找到该函数的源时，这才有可能；  出于这个原因，在加载 Edebug 之后，eval-region 会记录它评估的每个定义的位置，即使没有检测它。  另请参阅 i 命令（请参阅 Jumping），它在检测函数后进入调用。

Edebug 知道如何检测所有标准的特殊形式、带有表达式参数的交互式形式、匿名 lambda 表达式和其他定义形式。  但是，Edebug 无法自行确定用户定义的宏将如何处理宏调用的参数，因此您必须使用 Edebug 规范提供该信息；  有关详细信息，请参阅 Edebug 和宏。

当 Edebug 即将在会话中第一次检测代码时，它会运行钩子 edebug-setup-hook，然后将其设置为 nil。  您可以使用它来加载与您正在使用的包关联的 Edebug 规范，但仅限于使用 Edebug 时。

如果 Edebug 在检测时检测到语法错误，它将指向错误代码并发出无效读取语法错误信号。  例子：

    error→ Invalid read syntax: "Expected lambda expression"

这种检测失败的一个潜在原因是 Emacs 还不知道某些宏定义。  要解决此问题，请加载定义您将要检测的函数的文件。

要从定义中删除插桩，只需以不插桩的方式重新评估其定义。  有两种从不检测表单的方法：从带有 load 的文件，以及从带有 eval-expression (M-:) 的 minibuffer。

从定义中删除检测的另一种方法是使用 edebug-remove-instrumentation 命令。  它还允许从已检测的所有内容中删除检测。

有关 Edebug 内部可用的其他评估功能，请参阅评估。



### TODO 19.2.3 Edebug 执行模式

Edebug 支持多种执行模式来运行您正在调试的程序。  我们将这些替代方案称为 Edebug 执行模式；  不要将它们与主要或次要模式混淆。  当前的 Edebug 执行模式决定了 Edebug 在停止前继续执行多远——例如，它是在每个停止点停止，还是继续到下一个断点——以及 Edebug 在停止前显示评估进度的程度。

通常，您通过键入命令以某种模式继续程序来指定 Edebug 执行模式。  这是这些命令的表格；  除了 S 之外的所有程序都恢复执行，至少在一定距离内。

    S

停止：不再执行任何程序，而是等待更多的 Edebug 命令（edebug-stop）。

    SPC

Step：在遇到的下一个停止点停止（edebug-step-mode）。

    n

Next：在表达式之后遇到的下一个停止点停止（edebug-next-mode）。  另请参阅 Jumping 中的 edebug-forward-sexp。

    t

跟踪：在每个 Edebug 停止点（edebug-trace-mode）暂停（通常为一秒）。

    T

快速跟踪：在每个停止点更新显示，但实际上并不暂停（edebug-Trace-fast-mode）。

    g

Go：运行到下一个断点（edebug-go-mode）。  请参阅 Edebug 断点。

    c

继续：在每个断点处暂停一秒，然后继续（edebug-continue-mode）。

    C

快速继续：将点移动到每个断点，但不要暂停（edebug-Continue-fast-mode）。

    G

Go non-stop：忽略断点（edebug-Go-nonstop-mode）。  您仍然可以通过键入 S 或任何编辑命令来停止程序。

通常，上述列表中较早的执行模式比列表中较晚的模式运行程序更慢或停止得更快。

当您进入一个新的 Edebug 级别时，Edebug 通常会在它遇到的第一个检测函数处停止。  如果您希望只在断点处停止，或者根本不停止（例如，在收集覆盖率数据时），请将 edebug-initial-mode 的值从其默认步骤更改为 go、Go-nonstop 或其其中之一其他值（请参阅 Edebug 选项）。  您可以使用 Cx Ca Cm (edebug-set-initial-mode) 轻松完成此操作：

    Command: edebug-set-initial-mode ¶

此命令绑定到 Cx Ca Cm，设置 edebug-initial-mode。  它会提示您输入一个键来指示模式。  您应该输入上面列出的八个键之一，用于设置相应的模式。

请注意，您可能会多次重新输入相同的 Edebug 级别，例如，如果从一个命令多次调用检测函数。

在执行或跟踪时，您可以通过键入任何 Edebug 命令来中断执行。  Edebug 在下一个停止点停止程序，然后执行您键入的命令。  例如，在执行期间键入 t 会在下一个停止点切换到跟踪模式。  您可以使用 S 停止执行，而无需执行任何其他操作。

如果您的函数碰巧读取输入，则您键入的旨在中断执行的字符可能会被该函数读取。  您可以通过注意程序何时需要输入来避免这种意外结果。

包含本节中的命令的键盘宏不完全起作用：退出 Edebug 以恢复程序，失去对键盘宏的跟踪。  这不容易解决。  此外，在 Edebug 外部定义或执行键盘宏不会影响 Edebug 内部的命令。  这通常是一个优势。  另请参阅 Edebug 选项中的 edebug-continue-kbd-macro 选项。

    User Option: edebug-sit-for-seconds ¶

此选项指定在跟踪模式或继续模式下执行步骤之间等待的秒数。  默认值为 1 秒。



### TODO 19.2.4 跳跃

本节中描述的命令会一直执行，直到它们到达指定的位置。  除了我做一个临时断点来建立停止的地方，然后切换到 go 模式。  在预期停止点之前到达的任何其他断点也将停止执行。  有关断点的详细信息，请参阅 Edebug Breakpoints。

在非本地退出的情况下，这些命令可能无法按预期工作，因为这可以绕过您希望程序停止的临时断点。

    h

前往点所在位置附近的停止点 (edebug-goto-here)。

    f

为一个表达式运行程序 (edebug-forward-sexp)。

    o

运行程序直到包含的 sexp 结束（edebug-step-out）。

    i

点后单步执行表单调用的函数或宏（edebug-step-in）。

h 命令使用临时断点继续到点的当前位置或之后的停止点。

f 命令在一个表达式上向前运行程序。  更准确地说，它在 forward-sexp 将到达的位置设置一个临时断点，然后在 go 模式下执行，以便程序将在断点处停止。

使用前缀参数 n，临时断点放置在点外 n 秒。  如果包含列表在 n 多个元素之前结束，则停止位置在包含表达式之后。

您必须检查 forward-sexp 找到的位置是否是程序真正到达的位置。  例如，在条件下，这可能不是真的。

为灵活起见， f 命令从 point 开始执行 forward-sexp，而不是在 stop 点。  如果要从当前停止点执行一个表达式，首先键入 w (edebug-where) 将点移动到那里，然后键入 f。

o 命令从表达式继续。  它在 sexp 包含点的末尾放置一个临时断点。  如果包含的 sexp 本身是一个函数定义，则 o 会一直持续到定义中的最后一个 sexp 之前。  如果那是您现在所在的位置，它会从函数返回然后停止。  换句话说，该命令不会退出当前正在执行的函数，除非您位于最后一个 sexp 之后。

通常，h、f 和 o 命令会显示“Break”并暂停 edebug-sit-for-seconds，然后再显示刚刚评估的表单的结果。  您可以通过将 edebug-sit-on-break 设置为 nil 来避免这种暂停。  请参阅 Edebug 选项。

i 命令在点之后进入由列表形式调用的函数或宏，并在其第一个停止点停止。  请注意，表格不必是即将被评估的表格。  但是如果表单是一个即将被评估的函数调用，请记住在评估任何参数之前使用此命令，否则为时已​​晚。

i 命令检测它应该进入的函数或宏，如果它还没有检测的话。  这很方便，但请记住，除非您明确安排对它进行取消检测，否则该函数或宏将保持检测。



### TODO 19.2.5 其他 Edebug 命令

此处描述了一些杂项 Edebug 命令。

    ?

显示 Edebug 的帮助信息 (edebug-help)。

    a

    C-]

中止一个级别回到上一个命令级别（中止递归编辑）。

    q

返回到顶级编辑器命令循环（顶级）。  这将退出所有递归编辑级别，包括所有级别的 Edebug 活动。  但是，使用 unwind-protect 或条件案例形式保护的检测代码可能会恢复调试。

    Q

像 q，但即使是受保护的代码也不要停止（edebug-top-level-nonstop）。

    r

在回显区域重新显示最近已知的表达式结果 (edebug-previous-result)。

    d

显示回溯，为了清楚起见，不包括 Edebug 自己的函数（edebug-pop-to-backtrace）。

请参阅 Backtraces，了解对回溯和对其起作用的命令的描述。

如果您想在回溯中查看 Edebug 的功能，请使用 Mx edebug-backtrace-show-instrumentation。  要再次隐藏它们，请使用 Mx edebug-backtrace-hide-instrumentation。

如果回溯帧以“>”开头，则意味着 Edebug 知道该帧的源代码所在的位置。  使用 s 跳转到当前帧的源代码。

当您继续执行时，回溯缓冲区会自动终止。

您可以从 Edebug 调用以递归方式再次激活 Edebug 的命令。  每当 Edebug 处于活动状态时，您可以使用 q 退出到顶层或使用 C-] 中止一个递归编辑级别。  您可以使用 d 显示所有未决评估的回溯。



### TODO 19.2.6 休息

Edebug 的步进模式在到达下一个停止点时停止执行。  一旦 Edebug 开始执行，还有其他三种方法可以停止它：断点、全局断点条件和源断点。

1.  TODO 19.2.6.1 调试断点

    在使用 Edebug 时，您可以在您正在测试的程序中指定断点：这些是应该停止执行的地方。  您可以在任何停止点设置断点，如使用 Edebug 中所定义。  对于设置和取消设置断点，受影响的停止点是源代码缓冲区中的第一个或之后的停止点。  以下是断点的 Edebug 命令：
    
        b
    
    在点处或之后的停止点处设置断点 (edebug-set-breakpoint)。  如果使用前缀参数，断点是临时的——它在第一次停止程序时关闭。  具有 edebug-enabled-breakpoint 或 edebug-disabled-breakpoint 面的覆盖被放置在断点处。
    
        u
    
    在点 (edebug-unset-breakpoint) 处或之后的停止点处取消设置断点（如果有）。
    
        U
    
    取消设置当前表单中的任何断点 (edebug-unset-breakpoints)。
    
        D
    
    切换是否禁用点附近的断点 (edebug-toggle-disable-breakpoint)。  如果断点是有条件的并且需要一些工作来重新创建条件，则此命令非常有用。
    
        x condition RET
    
    设置一个条件断点，仅当评估条件产生非零值时才停止程序（edebug-set-conditional-breakpoint）。  使用前缀参数，断点是临时的。
    
        B
    
    将点移动到当前定义中的下一个断点 (edebug-next-breakpoint)。
    
    在 Edebug 中，您可以使用 b 设置断点并使用 u 取消设置。  首先将点移动到您选择的 Edebug 停止点，然后键入 b 或 u 在此处设置或取消设置断点。  取消设置没有设置的断点无效。
    
    重新评估或重新配置定义会删除其所有先前的断点。
    
    每次程序到达那里时，条件断点都会测试一个条件。  由于评估条件而发生的任何错误都将被忽略，就好像结果为零一样。  要设置条件断点，请使用 x，并在 minibuffer 中指定条件表达式。  在具有先前建立的条件断点的停止点处设置条件断点会将先前的条件表达式放在迷你缓冲区中，以便您可以对其进行编辑。
    
    您可以通过在命令中使用前缀参数来设置条件断点或无条件断点以设置断点。  当临时断点停止程序时，它会自动取消设置。
    
    Edebug 总是在断点处停止或暂停，除非 Edebug 模式是 Go-nonstop。  在这种模式下，它完全忽略断点。
    
    要找出断点的位置，请使用 B 命令，该命令将点移动到同一函数内的下一个断点，如果没有后续断点，则移动到第一个断点。  该命令不会继续执行——它只是在缓冲区中移动点。

2.  TODO 19.2.6.2 全局中断条件

    全局中断条件在满足指定条件时停止执行，无论发生在何处。  Edebug 在每个停止点评估全局中断条件；  如果它的计算结果为非零值，则执行停止或暂停，具体取决于执行模式，就好像遇到了断点一样。  如果评估条件出错，则执行不会停止。
    
    条件表达式存储在 edebug-global-break-condition 中。  您可以在 Edebug 处于活动状态时使用源代码缓冲区中的 X 命令指定新表达式，或者只要加载了 Edebug (edebug-set-global-break-condition)，就可以随时从任何缓冲区中使用 Cx XX。
    
    全局中断条件是查找代码中某个事件发生位置的最简单方法，但它会使代码运行得更慢。  所以你应该在不使用它时将条件重置为零。

3.  TODO 19.2.6.3 源断点

    每次重新设置定义时，都会忘记定义中的所有断点。  如果你想创建一个不会被遗忘的断点，你可以编写一个源断点，它只是在你的源代码中调用函数 edebug。  当然，您可以将这样的调用设为有条件的。  例如，在 fac 函数中，您可以如下所示插入第一行，以在参数达到零时停止：
    
        (defun fac (n)
          (if (= n 0) (edebug))
          (if (< 0 n)
              (* n (fac (1- n)))
            1))
    
    当检测 fac 定义并调用函数时，对 edebug 的调用充当断点。  根据执行模式，Edebug 会在那里停止或暂停。
    
    如果调用 edebug 时没有执行任何检测代码，则该函数调用 debug。



### TODO 19.2.7 捕获错误

Emacs 通常在发出错误信号且未使用条件案例处理时显示错误消息。  当 Edebug 处于活动状态并执行检测代码时，它通常会响应所有未处理的错误。  您可以使用选项 edebug-on-error 和 edebug-on-quit 自定义它；  请参阅 Edebug 选项。

当 Edebug 响应错误时，它会显示错误之前遇到的最后一个停止点。  这可能是调用未检测的函数的位置，并且错误实际发生在该位置。  对于未绑定的变量错误，最后一个已知的停止点可能与有问题的变量引用相距甚远。  在这种情况下，您可能希望显示完整的回溯（参见 Miscellaneous Edebug Commands）。

如果您在 Edebug 处于活动状态时更改 debug-on-error 或 debug-on-quit，这些更改将在 Edebug 变为非活动状态时被遗忘。  此外，在 Edebug 的递归编辑期间，这些变量被绑定到它们在 Edebug 之外的值。



### TODO 19.2.8 调试视图

这些 Edebug 命令让您可以查看缓冲区和窗口状态的各个方面，就像它们在进入 Edebug 之前一样。  外部窗口配置是在 Edebug 之外有效的窗口和内容的集合。

    P

    v

切换到查看外部窗口配置 (edebug-view-outside)。  键入 Cx X w 返回 Edebug。

    p

临时显示外部当前缓冲区，点位于其外部位置（edebug-bounce-point），暂停一秒钟，然后返回 Edebug。  使用前缀参数 n，改为暂停 n 秒。

    w

将点移回源代码缓冲区（edebug-where）中的当前停止点。

如果您在显示相同缓冲区的不同窗口中使用此命令，则将来将使用该窗口来显示当前定义。

    W

切换 Edebug 是否保存和恢复外部窗口配置 (edebug-toggle-save-windows)。

使用前缀参数，W 仅切换所选窗口的保存和恢复。  要指定不显示源代码缓冲区的窗口，您必须使用全局键盘映射中的 Cx XW。

您可以使用 v 查看外部窗口配置，或者使用 p 弹跳到当前缓冲区中的点，即使它没有正常显示。

移动点后，您不妨跳回停止点。  您可以使用来自源代码缓冲区的 w 来做到这一点。  您可以使用 Cx X w 从任何缓冲区跳回源代码缓冲区中的停止点。

每次您使用 W 关闭保存时，Edebug 都会忘记保存的外部窗口配置——因此即使您重新打开保存，当前窗口配置在您下次退出 Edebug 时（通过继续程序）保持不变。  然而，\*edebug\* 和 **edebug-trace** 的自动重新显示可能与您希望看到的缓冲区冲突，除非您打开了足够多的窗口。



### TODO 19.2.9 评估

在 Edebug 中，您可以评估表达式，就像 Edebug 没有运行一样。  Edebug 试图对表达式的评估和打印不可见。  除了 Edebug 显式保存和恢复的数据更改之外，导致副作用的表达式的评估将按预期工作。  有关此过程的详细信息，请参阅外部环境。

    C-x C-e

在 Edebug 之外的上下文中计算表达式 exp (edebug-eval-expression)。  也就是说，Edebug 会尽量减少对评估的干扰。

    M-: exp RET

在 Edebug 本身的上下文中计算表达式 exp (eval-expression)。

    e exp RET

在 Edebug 之外的上下文 (edebug-eval-last-sexp) 中计算点之前的表达式。  使用零前缀参数 (Cu 0 Cx Ce)，不要缩短长项目（如字符串和列表）。

Edebug 支持对包含对由 cl.el 中的以下构造创建的词法绑定符号的引用的表达式求值：lexical-let、macrolet 和 symbol-macrolet。



### TODO 19.2.10 评估列表缓冲区

您可以使用名为 **edebug** 的评估列表缓冲区以交互方式评估表达式。  您还可以设置每次 Edebug 更新显示时自动评估的表达式评估列表。

    E

切换到评估列表缓冲区 **edebug** (edebug-visit-eval-list)。

在 **edebug** 缓冲区中，您可以使用 Lisp 交互模式的命令（参见 GNU Emacs 手册中的 Lisp 交互）以及这些特殊命令：

    C-j

在外部上下文中评估点之前的表达式，并将值插入缓冲区（edebug-eval-print-last-sexp）。  前缀参数为零 (Cu 0 Cj) 时，不要缩短长项目（如字符串和列表）。

    C-x C-e

在 Edebug 之外的上下文 (edebug-eval-last-sexp) 中计算点之前的表达式。

    C-c C-u

从缓冲区的内容（edebug-update-eval-list）构建一个新的评估列表。

    C-c C-d

删除该点所在的评估列表组 (edebug-delete-eval-item)。

    C-c C-w

在当前停止点（edebug-where）切换回源代码缓冲区。

您可以使用 Cj 或 Cx Ce 在评估列表窗口中评估表达式，就像在 **scratch** 中一样；  但它们是在 Edebug 之外的上下文中评估的。

当您继续执行时，您以交互方式输入的表达式（及其结果）将丢失；  但是您可以设置一个评估列表，其中包含每次执行停止时要评估的表达式。

为此，请在评估列表缓冲区中写入一个或多个评估列表组。  评估列表组由一个或多个 Lisp 表达式组成。  组由注释行分隔。

命令 Cc Cu (edebug-update-eval-list) 重建评估列表，扫描缓冲区并使用每个组的第一个表达式。  （想法是组的第二个表达式是先前计算和显示的值。）

Edebug 的每个条目通过在缓冲区中插入每个表达式，然后是其当前值来重新显示评估列表。  它还插入注释行，以便每个表达式成为自己的组。  因此，如果您在不更改缓冲区文本的情况下再次键入 Cc Cu，则评估列表实际上不会更改。

如果在评估列表中的评估期间发生错误，则错误消息将显示在字符串中，就好像它是结果一样。  因此，使用当前无效变量的表达式不会中断您的调试。

以下是添加了几个表达式后评估列表窗口的外观示例：

    (current-buffer)
    #<buffer *scratch*>
    ;---------------------------------------------------------------
    (selected-window)
    #<window 16 on *scratch*>
    ;---------------------------------------------------------------
    (point)
    196
    ;---------------------------------------------------------------
    bad-var
    "Symbol's value as variable is void: bad-var"
    ;---------------------------------------------------------------
    (recursion-depth)
    0
    ;---------------------------------------------------------------
    this-command
    eval-last-sexp
    ;---------------------------------------------------------------

要删除一个组，将点移入其中并键入 Cc Cd，或者简单地删除该组的文本并使用 Cc Cu 更新评估列表。  要将新表达式添加到评估列表，请在合适的位置插入表达式，插入新的注释行，然后键入 Cc Cu。  您无需在注释行中插入破折号——其内容无关紧要。

选择 **edebug** 后，您可以使用 Cc Cw 返回到源代码缓冲区。  **edebug** 缓冲区在您继续执行时被终止，并在下次需要时重新创建。



### TODO 19.2.11 在 Edebug 中打印

如果您的程序中的表达式产生一个包含循环列表结构的值，当 Edebug 尝试打印它时，您可能会遇到错误。

处理循环结构的一种方法是设置打印长度或打印级别以截断打印。  Edebug 为你做这件事；  它将 print-length 和 print-level 绑定到变量 edebug-print-length 和 edebug-print-level 的值（只要它们具有非 nil 值）。  请参阅影响输出的变量。

    User Option: edebug-print-length ¶

如果非零，Edebug 在打印结果时将 print-length 绑定到这个值。  默认值为 50。

    User Option: edebug-print-level ¶

如果非 nil，Edebug 会在打印结果时将 print-level 绑定到该值。  默认值为 50。

您还可以通过将 print-circle 绑定到非 nil 值来打印循环结构和共享元素的结构。

下面是一个创建循环结构的代码示例：

    (setq a (list 'x 'y))
    (setcar a a)

如果 print-circle 不为零，则打印函数（例如，prin1）会将 a 打印为 '#1=(#1# y)'。  '#1=' 符号用标签 '1' 标记紧随其后的结构，而 '#1#' 符号引用先前标记的结构。  此表示法用于列表或向量的任何共享元素。

    User Option: edebug-print-circle ¶

如果非零，Edebug 在打印结果时将 print-circle 绑定到这个值。  默认值为 t。

有关如何自定义打印的更多详细信息，请参阅输出函数。



### TODO 19.2.12 跟踪缓冲区

Edebug 可以记录执行跟踪，将其存储在名为 **edebug-trace** 的缓冲区中。  这是函数调用和返回的日志，显示函数名称及其参数和值。  要启用跟踪记录，请将 edebug-trace 设置为非零值。

创建跟踪缓冲区与使用跟踪执行模式不同（请参阅 Edebug 执行模式）。

启用跟踪记录后，每个函数入口和出口都会向跟踪缓冲区添加行。  函数入口记录由 '::::{' 组成，后跟函数名称和参数值。  函数退出记录由 '::::}' 组成，后跟函数名和函数结果。

条目中 ':' 的数量表示其递归深度。  您可以使用跟踪缓冲区中的大括号来查找匹配的函数调用的开头或结尾。

您可以通过重新定义函数 edebug-print-trace-before 和 edebug-print-trace-after 来自定义函数进入和退出的跟踪记录。

    Macro: edebug-tracing string body… ¶

此宏请求有关执行主体表单的附加跟踪信息。  参数字符串指定要放入跟踪缓冲区的文本，位于“{”或“}”之后。  所有参数都被评估，并且 edebug-tracing 返回正文中最后一个表单的值。

    Function: edebug-trace format-string &rest format-args ¶

此函数在跟踪缓冲区中插入文本。  它使用 (apply 'format format-string format-args) 计算文本。  它还将换行符附加到单独的条目。

无论何时调用 edebug-tracing 和 edebug-trace 都会在跟踪缓冲区中插入行，即使 Edebug 未处于活动状态。  将文本添加到跟踪缓冲区还会滚动其窗口以显示最后插入的行。



### TODO 19.2.13 覆盖测试

Edebug 提供基本的覆盖测试和执行频率的显示。

覆盖测试通过将每个表达式的结果与之前的结果进行比较来工作；  如果自从您在当前 Emacs 会话中开始测试覆盖率以来，程序中的每个表单都返回了两个不同的值，则认为它已被覆盖。  因此，要对您的程序进行覆盖测试，请在各种条件下执行它并注意其行为是否正确；  当您尝试了足够多的不同条件以使每个表单返回两个不同的值时，Edebug 会告诉您。

覆盖测试会使执行速度变慢，因此只有在 edebug-test-coverage 不为零时才会执行。  对检测函数的所有执行执行频率计数，即使执行模式是 Go-nonstop，也不管是否启用了覆盖测试。

使用 Cx X = (edebug-display-freq-count) 显示定义的覆盖信息和频率计数。  Just = (edebug-temp-display-freq-count) 暂时显示相同的信息，直到您键入另一个键。

    Command: edebug-display-freq-count ¶

此命令显示当前定义的每一行的频率计数数据。

它在每行代码之后插入频率计数作为注释行。  您可以使用一个撤消命令撤消所有插入。  计数出现在表达式之前的 '(' 或表达式之后的 ')' 下，或变量的最后一个字符上。  为了简化显示，如果计数等于同一行上先前表达式的计数，则不显示计数。

表达式计数后面的字符“=”表示该表达式每次计算时都返回相同的值。  换句话说，它还没有被覆盖用于覆盖测试目的。

要清除定义的频率计数和覆盖数据，只需使用 eval-defun 重新设置即可。

例如，在使用源断点评估 (fac 5) 并将 edebug-test-coverage 设置为 t 后，到达断点时，频率数据如下所示：

    (defun fac (n)
      (if (= n 0) (edebug))
    ;#6           1      = =5
      (if (< 0 n)
    ;#5         =
          (* n (fac (1- n)))
    ;#    5               0
        1))
    ;#   0

注释行显示 fac 被调用了 6 次。  第一个 if 语句返回 5 次，每次都返回相同的结果；  第二个 if 的条件也是如此。  fac 的递归调用根本没有返回。



### TODO 19.2.14 外部环境

Edebug 试图对您正在调试的程序透明，但它并没有完全成功。  当您使用 e 或评估列表缓冲区评估表达式时，Edebug 也会通过临时恢复外部上下文来尝试保持透明。  本节准确解释了 Edebug 恢复了什么上下文，以及 Edebug 如何无法完全透明。

1.  TODO 19.2.14.1 检查是否停止

    每当进入 Edebug 时，它甚至需要在决定是否制作跟踪信息或停止程序之前保存和恢复某些数据。
    
    max-lisp-eval-depth（参见 Eval）和 max-specpdl-size（参见局部变量）都增加以减少 Edebug 对堆栈的影响。  但是，在使用 Edebug 时，您仍然可能会用完堆栈空间。  如果 Edebug 达到包含非常大的引用列表的递归深度检测代码的限制，您还可以扩大 edebug-max-depth 的值。
    键盘宏执行的状态被保存和恢复。  当 Edebug 处于活动状态时，executing-kbd-macro 绑定为 nil，除非 edebug-continue-kbd-macro 为非 nil。

2.  TODO 19.2.14.2 调试显示更新

    当 Edebug 需要显示某些东西时（例如，在跟踪模式下），它会从 Edebug 外部保存当前窗口配置（请参阅窗口配置）。  当您退出 Edebug 时，它会恢复之前的窗口配置。
    
    Emacs 只有在暂停时才会重新显示。  通常，当您继续执行时，程序会在断点处或单步执行后重新进入 Edebug，而不会在其间暂停或读取输入。  在这种情况下，Emacs 永远没有机会重新显示外部配置。  因此，您所看到的窗口配置与上次 Edebug 处于活动状态时相同，没有中断。
    
    进入 Edebug 以显示某些内容也会保存和恢复以下数据（尽管如果出现错误或退出信号，其中一些是故意不恢复的）。
    
    保存和恢复当前缓冲区是哪个缓冲区，以及当前缓冲区中的点和标记的位置。
    如果 edebug-save-windows 不为零（请参阅 Edebug 选项），则会保存和恢复外部窗口配置。
    
    窗口配置不会在错误或退出时恢复，但即使在错误或退出时会重新选择外部选定的窗口，以防保存行程处于活动状态。  如果 edebug-save-windows 的值为列表，则仅保存和恢复列出的窗口。
    
    但是，源代码缓冲区的窗口开始和水平滚动不会恢复，因此显示在 Edebug 中保持一致。
    如果 edebug-save-displayed-buffer-points 不为零，则保存和恢复每个显示缓冲区中的点值。
    变量 overlay-arrow-position 和 overlay-arrow-string 被保存和恢复，因此您可以安全地从同一缓冲区中其他地方的递归编辑调用 Edebug。
    cursor-in-echo-area 本地绑定到 nil 以便光标显示在窗口中。

3.  TODO 19.2.14.3 Edebug 递归编辑

    当进入 Edebug 并实际从用户读取命令时，它会保存（并稍后恢复）这些附加数据：
    
    当前匹配数据。  请参阅匹配数据。
    变量 last-command、this-command、last-command-event、last-input-event、last-event-frame、last-nonmenu-event 和 track-mouse。  Edebug 中的命令不会影响 Edebug 之外的这些变量。
    
    在 Edebug 中执行命令可以更改 this-command-keys 返回的键序列，并且无法从 Lisp 中重置键序列。
    
    Edebug 无法保存和恢复 unread-command-events 的值。  在此变量具有重要值时输入 Edebug 可能会干扰您正在调试的程序的执行。
    在 Edebug 中执行的复杂命令被添加到变量 command-history。  在极少数情况下，这可能会改变执行。
    在 Edebug 内，递归深度似乎比 Edebug 外的递归深度深一。  自动更新的评估列表窗口并非如此。
    递归编辑将标准输出和标准输入绑定为 nil，但 Edebug 会在评估期间临时恢复它们。
    键盘宏定义的状态被保存和恢复。  当 Edebug 处于活动状态时，defining-kbd-macro 绑定到 edebug-continue-kbd-macro。



### TODO 19.2.15 调试和宏

为了使 Edebug 正确地检测调用宏的表达式，需要格外小心。  本小节解释了细节。

1.  TODO 19.2.15.1 检测宏调用

    当 Edebug 检测调用 Lisp 宏的表达式时，它需要有关宏的附加信息才能正确完成工作。  这是因为没有先验方法来判断宏调用的哪些子表达式是要评估的形式。  （评估可能会在宏体中显式发生，或者在评估结果扩展时，或者稍后的任何时间。）
    
    因此，您必须为 Edebug 将遇到的每个宏定义一个 Edebug 规范，以解释对该宏的调用格式。  为此，请将调试声明添加到宏定义中。  这是一个简单的示例，显示了示例宏的规范（请参阅重复评估宏参数）。
    
        (defmacro for (var from init to final do &rest body)
          "Execute a simple \"for\" loop.
        For example, (for i from 1 to 10 do (print i))."
          (declare (debug (symbolp "from" form "to" form "do" &rest form)))
          ...)
    
    Edebug 规范说明了对宏的调用的哪些部分是要评估的表单。  对于简单的宏，规范通常看起来与宏定义的形式参数列表非常相似，但规范比宏参数更通用。  有关声明形式的更多说明，请参见定义宏。
    
    当您检测代码时，请注意确保 Edebug 知道规范。  如果您正在检测使用在另一个文件中定义的宏的函数，您可能首先需要评估包含您的函数的文件中的 require 表单，或者显式加载包含宏的文件。  如果宏的定义由 eval-when-compile 包装，您可能需要对其求值。
    
    您还可以使用 def-edebug-spec 将宏定义与宏定义分开定义 edebug 规范。  对于 Lisp 中的宏定义，添加调试声明是首选且更方便，但 def-edebug-spec 可以为 C 中实现的特殊形式定义 Edebug 规范。
    
        Macro: def-edebug-spec macro specification ¶
    
    指定调用宏的哪些表达式是要评估的形式。  规范应该是 Edebug 规范。  两个参数都没有被评估。
    
    宏参数实际上可以是任何符号，而不仅仅是宏名称。
    
    下表列出了规范的可能性以及每种可能性如何指导参数的处理。
    
        t
    
    所有论点都用于评估。  这是（身体）的缩写。
    
        a symbol
    
    该符号必须有一个 Edebug 规范，以替代使用。  重复这种间接方式，直到找到另一种规范。  这允许您从另一个宏继承规范。
    
        a list
    
    列表的元素描述了调用表单的参数类型。  规格列表的可能元素将在以下部分中描述。
    
    如果宏没有 Edebug 规范，无论是通过调试声明还是通过 def-edebug-spec 调用，变量 edebug-eval-macro-args 都会发挥作用。
    
        User Option: edebug-eval-macro-args ¶
    
    这控制了 Edebug 处理没有明确 Edebug 规范的宏参数的方式。  如果它是 nil（默认值），则不会对任何参数进行评估。  否则，所有参数都会被检测。

2.  TODO 19.2.15.2 规格表

    如果宏调用的某些参数被评估而其他参数不被评估，则 Edebug 规范需要规范列表。  规范列表中的一些元素匹配一个或多个参数，但其他元素修改所有后续元素的处理。  后者称为规范关键字，是以“&”开头的符号（例如 &optional）。
    
    规范列表可能包含子列表，这些子列表匹配本身就是列表的参数，或者它可能包含用于分组的向量。  子列表和组因此将规范列表细分为层次结构。  规范关键字仅适用于包含它们的子列表或组的其余部分。
    
    当规范列表涉及替代或重复时，将其与实际的宏调用进行匹配可能需要回溯。  有关更多详细信息，请参阅规范中的回溯。
    
    Edebug 规范提供了正则表达式匹配的强大功能，以及一些上下文无关的语法结构：使用平衡括号匹配子列表、表单的递归处理以及通过间接规范进行递归。
    
    以下是规范列表的可能元素及其含义的表格（参见规范示例，参考示例）：
    
        sexp
    
    单个未计算的 Lisp 对象，未检测。
    
        form
    
    单个评估的表达式，它被检测。  如果您的宏在计算之前用 lambda 包装了表达式，请改用 def-form。  请参见下面的 def-form。
    
        place
    
    广义变量。  请参阅广义变量。
    
        body
    
    &rest 形式的缩写。  请参阅下面的 &rest。  如果您的宏在评估之前使用 lambda 包装其代码体，请改用 def-body。  请参阅下面的 def-body。
    
        lambda-expr
    
    没有引号的 lambda 表达式。
    
        &optional
    
    规格列表中​​的所有以下元素都是可选的；  一旦有一个不匹配，Edebug 就会在此级别停止匹配。
    
    要使几个元素成为可选元素，然后是非可选元素，请使用 [&optional specs&#x2026;]。  要指定几个元素必须全部匹配或不匹配，请使用 &optional [specs&#x2026;]。  请参阅 defun 示例。
    
        &rest
    
    规格列表中​​的所有以下元素重复零次或多次。  然而，在最后的重复中，如果表达式在匹配规范列表的所有元素之前用完，这不是问题。
    
    要仅重复几个元素，请使用 [&rest specs&#x2026;]。  要指定在每次重复时必须全部匹配的几个元素，请使用 &rest [specs&#x2026;]。
    
        &or
    
    规格列表中​​的以下每个元素都是一个替代项。  备选方案之一必须匹配，否则 &or 规范失败。
    
    &or 之后的每个列表元素都是一个替代项。  要将两个或多个列表元素组合为一个备选方案，请将它们括在 [&#x2026;] 中。
    
        &not
    
    下面的每个元素都被匹配为替代项，就像使用 &or 一样，但如果其中任何一个匹配，则说明失败。  如果它们都不匹配，则不匹配，但 &not 规范成功。
    
        &define
    
    表示规范是针对定义形式的。  Edebug 对定义表单的定义是包含一个或多个代码表单的表单，这些代码表单在定义表单执行后保存并稍后执行。
    
    定义表单本身没有被检测（也就是说，Edebug 不会在定义表单之前和之后停止），但它内部的表单通常会被检测。  &define 关键字应该是列表规范中的第一个元素。
    
        nil
    
    当当前参数列表级别没有更多参数匹配时，这是成功的；  否则失败。  请参阅子列表规范和反引号示例。
    
        gate ¶
    
    没有匹配的参数，但在匹配此级别的其余规范时禁用通过门的回溯。  这主要用于生成更具体的语法错误消息。  有关详细信息，请参阅规范中的回溯。  另请参见 let 示例。
    
        &error
    
    &error 后面应该跟一个字符串，一条错误消息，在 edebug-spec 中；  它中止检测，在 minibuffer 中显示消息。
    
        &interpose
    
    让函数控制剩余代码的解析。  它采用 &interpose spec fun args&#x2026; 的形式，这意味着 Edebug 将首先将 spec 与代码匹配，然后使用匹配 spec 的代码调用 fun，一个解析函数 pf，最后是 args&#x2026;。解析函数需要一个单个参数，指示用于解析剩余代码的规范列表。  它应该只被调用一次并返回 fun 预期返回的检测代码。  例如 (&interpose symbolp pcase&#x2013;match-pat-args) 匹配第一个元素是符号的 sexps，然后让 pcase&#x2013;match-pat-args 根据 pcase&#x2013;match-pat- 查找与该头部符号关联的规范args 并将它们传递给它作为参数接收的 pf。
    
        other-symbol ¶
    
    规范列表中的任何其他符号都可以是谓词或间接规范。
    
    如果符号具有 Edebug 规范，则此间接规范应该是用于代替符号的列表规范，或者是调用以处理参数的函数。  规范可以用 def-edebug-elem-spec 定义：
    
    功能：def-edebug-elem-spec 元素规范¶
    
    定义用于代替符号元素的规范。  规范必须是一个列表。
    
    否则，符号应该是谓词。  谓词与参数一起调用，如果谓词返回 nil，则规范失败并且参数不会被检测。
    
    一些合适的谓词包括 symbolp、integerp、stringp、vectorp 和 atom。
    
        [elements…] ¶
    
    元素向量将元素分组为单个组规范。  它的含义与向量无关。
    
        "string"
    
    参数应该是一个名为字符串的符号。  该规范等效于带引号的符号 'symbol，其中符号的名称是字符串，但首选字符串形式。
    
        (vector elements…)
    
    参数应该是一个向量，其元素必须与规范中的元素匹配。  请参阅反引号示例。
    
        (elements…)
    
    任何其他列表都是子列表规范，并且参数必须是其元素与规范元素匹配的列表。
    
    子列表规范可以是一个点列表，然后相应的列表参数可以是一个点列表。  或者，点列表规范的最后一个 CDR 可以是另一个子列表规范（通过分组或间接规范，例如 (spec . [(more specs…)])），其元素与非点列表参数匹配。  这在递归规范中很有用，例如在反引号示例中。  另请参阅上面对 nil 规范的描述以终止此类递归。
    
    请注意，写为 (specs . nil) 的子列表规范等价于 (specs)，并且 (specs . (sublist-elements&#x2026;)) 等价于 (specs sublist-elements&#x2026;)。
    
    以下是可能仅在 &define 之后出现的附加规范列表。  请参阅 defun 示例。
    
        &name
    
    从代码中提取当前定义表单的名称。  它采用 &name [prestring] spec [poststring] fun args&#x2026; 的形式，这意味着 Edebug 会将 spec 与代码匹配，然后调用 fun 并连接当前名称、args&#x2026;、prestring、匹配的代码规范和后字符串。  如果 fun 不存在，则默认为连接参数的函数（在前一个名称和新名称之间有一个 @）。
    
        name
    
    参数是一个符号，是定义形式的名称。  [&name symbolp] 的简写。
    
    定义表单不需要具有名称字段；  它可能有多个名称字段。
    
        arg
    
    参数是一个符号，是定义形式的参数的名称。  但是，不允许使用 lambda 列表关键字（以 '&' 开头的符号）。
    
        lambda-list ¶
    
    这匹配一个 lambda 列表——一个 lambda 表达式的参数列表。
    
        def-body
    
    参数是定义中的代码主体。  这就像上面描述的主体，但定义主体必须使用不同的 Edebug 调用来检测与定义关联的信息。  使用 def-body 作为定义中最高级别的表单列表。
    
        def-form
    
    参数是定义中的单一、最高级别的形式。  这类似于 def-body，除了它用于匹配单个表单而不是表单列表。  作为一种特殊情况，def-form 还意味着在执行表单时不输出跟踪信息。  请参阅交互式示例。

3.  TODO 19.2.15.3 规范中的回溯

    如果规范在某些时候无法匹配，这并不一定意味着会发出语法错误信号；  相反，将进行回溯，直到用尽所有替代方案。  最终，参数列表的每个元素都必须与规范中的某个元素匹配，并且规范中的每个必需元素都必须匹配某个参数。
    
    当检测到语法错误时，可能要到很久以后才会报告，在更高级别的替代方案已经用尽之后，并且该点距离真正的错误更远。  但是如果发生错误时禁用回溯，则可以立即报告。  请注意，在几种情况下，回溯也会自动重新启用；  当 &optional、&rest 或 &or 或在开始处理子列表、组或间接规范时建立新的替代方案时。  启用或禁用回溯的效果仅限于当前正在处理的级别的其余部分和较低级别。
    
    匹配任何表单规范（即表单、正文、def-form 和 def-body）时，将禁用回溯。  这些规范将匹配任何形式，因此任何错误都必须在形式本身而不是更高级别。
    
    在成功匹配带引号的符号、字符串规范或 &define 关键字后，回溯也被禁用，因为这通常表示已识别的构造。  但是，如果您有一组都以相同符号开头的替代构造，您通常可以通过将符号从替代中分解来解决此约束，例如 ["foo" &or [first case] [second case] .. .]。
    
    大多数需求都可以通过这两种自动禁用回溯的方式来满足，但有时通过使用门规范显式禁用回溯很有用。  当您知道没有更高的替代方案可以应用时，这很有用。  请参阅 let 规范的示例。

4.  TODO 19.2.15.4 规范示例

    通过研究此处提供的示例，可能更容易理解 Edebug 规范。
    
    考虑一个假设的宏 my-test-generator，它在提供的数据列表上运行测试。  尽管 Edebug 的默认行为是不将参数作为代码进行检测，但由 edebug-eval-macro-args 控制（请参阅检测宏调用），但显式记录参数是数据会很有用：
    
        (def-edebug-spec my-test-generator (&rest sexp))
    
    一个 let 特殊形式有一个绑定序列和一个主体。  每个绑定要么是一个符号，要么是一个带有符号和可选表达式的子列表。  在下面的规范中，请注意子列表内部的门，以防止在找到子列表后回溯。
    
        (def-edebug-spec let
          ((&rest
            &or symbolp (gate symbolp &optional form))
           body))
    
    Edebug 对 defun 以及相关的参数列表和交互规范使用以下规范。  有必要专门处理交互式表单，因为表达式参数实际上是在函数体之外评估的。  （defmacro 的规范与 defun 的规范非常相似，但允许声明语句。）
    
        
        
        (def-edebug-spec defun
          (&define name lambda-list
        	   [&optional stringp]   ; Match the doc string, if present.
        	   [&optional ("interactive" interactive)]
        	   def-body))
        
        (def-edebug-elem-spec 'lambda-list
          '(([&rest arg]
             [&optional ["&optional" arg &rest arg]]
             &optional ["&rest" arg]
             )))
        
        (def-edebug-elem-spec 'interactive
          '(&optional &or stringp def-form))    ; Notice: def-form
    
    下面的反引号规范说明了如何匹配点列表并使用 nil 来终止递归。  它还说明了如何匹配向量的分量。  （由 Edebug 定义的实际规范略有不同，并且不支持点列表，因为这样做会导致非常深的递归，可能会失败。）
    
        (def-edebug-spec \` (backquote-form))   ; Alias just for clarity.
        
        (def-edebug-elem-spec 'backquote-form
          '(&or ([&or "," ",@"] &or ("quote" backquote-form) form)
        	(backquote-form . [&or nil backquote-form])
        	(vector &rest backquote-form)
        	sexp))



### TODO 19.2.16 调试选项

这些选项会影响 Edebug 的行为：

    User Option: edebug-setup-hook ¶

在使用 Edebug 之前调用的函数。  每次将其设置为新值时，Edebug 都会调用这些函数一次，然后将 edebug-setup-hook 重置为 nil。  您可以使用它来加载与您正在使用的包关联的 Edebug 规范，但前提是您也使用 Edebug。  请参阅 Edebug 检测。

    User Option: edebug-all-defs ¶

如果这是非零，则对定义形式（如 defun 和 defmacro）的正常评估会为 Edebug 提供工具。  这适用于 eval-defun、eval-region、eval-buffer 和 eval-current-buffer。

使用命令 Mx edebug-all-defs 切换此选项的值。  请参阅 Edebug 检测。

    User Option: edebug-all-forms ¶

如果这不是 nil，则命令 eval-defun、eval-region、eval-buffer 和 eval-current-buffer 仪器所有形式，即使是那些没有定义任何东西的形式。  这不适用于 minibuffer 中的加载或评估。

使用命令 Mx edebug-all-forms 切换此选项的值。  请参阅 Edebug 检测。

    User Option: edebug-eval-macro-args ¶

当这是非零时，所有宏参数都将在生成的代码中进行检测。  对于任何宏，调试声明都会覆盖此选项。  因此，要为某些已评估而有些未评估参数的宏指定异常，请使用调试声明指定 Edebug 表单规范。

    User Option: edebug-save-windows ¶

如果这是非零，Edebug 保存并恢复窗口配置。  这需要一些时间，所以如果您的程序不关心窗口配置会发生什么，最好将此变量设置为 nil。

如果该值为列表，则仅保存和恢复列出的窗口。

您可以在 Edebug 中使用 W 命令以交互方式更改此变量。  请参阅 Edebug 显示更新。

    User Option: edebug-save-displayed-buffer-points ¶

如果这是非零，Edebug 将保存并恢复所有显示缓冲区中的点。

如果您正在调试更改显示在非选定窗口中的缓冲区的点的代码，则必须在其他缓冲区中保存和恢复点。  如果 Edebug 或用户随后选择了窗口，则该缓冲区中的点将移动到窗口的点值。

在所有缓冲区中保存和恢复点很昂贵，因为它需要选择每个窗口两次，所以只有在需要时才启用它。  请参阅 Edebug 显示更新。

    User Option: edebug-initial-mode ¶

如果此变量非零，则它指定 Edebug 首次激活时的初始执行模式。  可能的值是 step、next、go、Go-nonstop、trace、Trace-fast、continue 和 Continue-fast。

默认值为步长。  该变量可以通过 Cx Ca Cm 交互设置（edebug-set-initial-mode）。  请参阅 Edebug 执行模式。

    User Option: edebug-trace ¶

如果这是非零，跟踪每个函数的进入和退出。  跟踪输出显示在名为 **edebug-trace** 的缓冲区中，每行一个函数入口或出口，按递归级别缩进。

另请参阅跟踪缓冲区中的 edebug-tracing。

    User Option: edebug-test-coverage ¶

如果非零，Edebug 测试所有被调试表达式的覆盖率。  请参阅覆盖测试。

    User Option: edebug-continue-kbd-macro ¶

如果非零，则继续定义或执行在 Edebug 之外执行的任何键盘宏。  谨慎使用它，因为它没有被调试。  请参阅 Edebug 执行模式。

    User Option: edebug-print-length ¶

如果非 nil，则在 Edebug 中打印结果的默认值 print-length。  请参阅影响输出的变量。

    User Option: edebug-print-level ¶

如果非 nil，则在 Edebug 中打印结果的默认值 print-level。  请参阅影响输出的变量。

    User Option: edebug-print-circle ¶

如果非 nil，则在 Edebug 中打印结果的 print-circle 的默认值。  请参阅影响输出的变量。

    User Option: edebug-unwrap-results ¶

如果非零，Edebug 会在显示表达式的结果时尝试删除它自己的任何检测。  这在调试表达式的结果本身是检测表达式的宏时是相关的。  作为一个非常人为的示例，假设示例函数 fac 已被检测，并考虑以下形式的宏：

    (defmacro test () "Edebug example."
      (if (symbol-function 'fac)
          …))

如果您对测试宏进行检测并单步执行，则默认情况下，符号函数调用的结果具有大量 edebug-after 和 edebug-before 形式，这可能会导致难以看到实际结果。  如果 edebug-unwrap-results 不为零，Edebug 会尝试从结果中删除这些形式。

    User Option: edebug-on-error ¶

如果 debug-on-error 以前为 nil，则 Edebug 将 debug-on-error 绑定到此值。  请参阅捕获错误。

    User Option: edebug-on-quit ¶

如果 debug-on-quit 以前为 nil，则 Edebug 将 debug-on-quit 绑定到此值。  请参阅捕获错误。

如果在 Edebug 处于活动状态时更改 edebug-on-error 或 edebug-on-quit 的值，则在下次通过新命令调用 Edebug 之前不会使用它们的值。

    User Option: edebug-global-break-condition ¶

如果非零，则在每个停止点测试的表达式。  如果结果非零，则中断。  错误被忽略。  请参阅全局中断条件。

    User Option: edebug-sit-for-seconds ¶

到达断点且执行模式为跟踪或继续时暂停的秒数。  请参阅 Edebug 执行模式。

    User Option: edebug-sit-on-break ¶

到达断点时是否暂停 edebug-sit-for-seconds。  设置为 nil 以防止暂停，非 nil 以允许它。

    User Option: edebug-behavior-alist ¶

默认情况下，此列表包含一个带有键 edebug 的条目和一个包含三个函数的列表，这些函数是插入检测代码中的函数的默认实现：edebug-enter、edebug-before 和 edebug-after。  要全局更改 Edebug 的行为，请修改默认条目。

Edebug 的行为也可以通过在此列表中添加一个条目来基于每个定义进行更改，其中包含您选择的键和三个功能。  然后将检测定义的 edebug-behavior 符号属性设置为新条目的键，Edebug 将为该定义调用新函数代替它自己的函数。

    User Option: edebug-new-definition-function ¶

在包装定义或闭包的主体后由 Edebug 运行的函数。  在 Edebug 初始化它自己的数据之后，这个函数被调用一个参数，即与定义相关的符号，它可能是实际定义的符号或由 Edebug 生成的符号。  此函数可用于设置由 Edebug 检测的每个定义的 edebug-behavior 符号属性。

    User Option: edebug-after-instrumentation-function ¶

要在使用之前检查或修改 Edebug 的检测，请将​​此变量设置为一个函数，该函数接受一个参数，一个检测的顶级表单，并返回相同或替换的表单，然后 Edebug 将使用它作为检测的最终结果.



## TODO 19.3 调试无效的 Lisp 语法

Lisp 阅读器报告无效语法，但不能说出真正的问题在哪里。  例如，在评估表达式时出现错误“解析期间文件结束”表示过多的开括号（或方括号）。  阅读器在文件末尾检测到这种不平衡，但它无法确定右括号应该在哪里。  同样，'Invalid read syntax: ")"' 表示多余的右括号或缺少左括号，但没有说明缺少的括号所属的位置。  那么，如何找到要更改的内容呢？

如果问题不只是括号的不平衡，一个有用的技术是在每个 defun 的开头尝试 CMe（end-of-defun，参见 GNU Emacs 手册中的 Defuns 移动），看看它是否到了那个地方那个defun似乎结束的地方。  如果没有，则该 defun 中存在问题。

然而，不匹配的括号是 Lisp 中最常见的语法错误，我们可以针对这些情况提供进一步的建议。  （此外，仅在启用 Show Paren 模式的代码中移动点可能会发现不匹配。）



### TODO 19.3.1 多余的开括号

第一步是找到不平衡的defun。  如果有多余的左括号，方法是转到文件末尾并键入 Cu CMu（向后列表，请参阅 The GNU Emacs Manual 中的通过 Parens 移动）。  这会将您移动到第一个不平衡的 defun 的开头。

下一步是准确确定问题所在。  除了研究程序之外，没有办法确定这一点，但现有的缩进通常是括号应该在哪里的线索。  使用此线索的最简单方法是使用 CMq 重新缩进（indent-pp-sexp，请参阅 GNU Emacs 手册中的多行缩进）并查看移动的内容。  但不要这样做！  继续阅读，首先。

在你这样做之前，确保 defun 有足够的右括号。  否则，CMq 将出现错误，或者将重新缩进文件的所有其余部分，直到结束。  所以移动到 defun 的末尾并在那里插入一个右括号。  不要使用 CMe (end-of-defun) 移动到那里，因为在 defun 平衡之前这也将无法工作。

现在您可以转到 defun 的开头并键入 CMq。  通常从某个点到函数结束的所有行都会向右移动。  在该点附近可能缺少右括号或多余的左括号。  （但是，不要假设这是真的；研究代码以确保。）一旦发现差异，使用 C-\_ 撤消 CMq（撤消），因为旧缩进可能适合预期的括号。

在你认为你已经解决了问题之后，再次使用 CMq。  如果旧缩进实际上适合括号的预期嵌套，并且您已放回这些括号，则 CMq 不应更改任何内容。



### TODO 19.3.2 多余的右括号

要处理多余的右括号，首先转到文件的开头，然后键入 Cu -1 CMu（参数为 -1 的向后列表）以查找第一个不平衡 defun 的结尾。

然后在 defun 的开头键入 CMf（forward-sexp，参见 GNU Emacs 手册中的表达式）找到实际匹配的右括号。  这会让你远离 defin 应该结束的地方。  您可能会在该附近找到一个虚假的右括号。

如果此时您没有发现问题，接下来要做的是在 defun 的开头键入 CMq (indent-pp-sexp)。  一系列行可能会向左移动；  如果是这样，则缺少的左括号或虚假的右括号可能在这些行的第一行附近。  （但是，不要假设这是真的；研究代码以确保。）一旦发现差异，使用 C-\_ 撤消 CMq（撤消），因为旧缩进可能适合预期的括号。

在你认为你已经解决了问题之后，再次使用 CMq。  如果旧的缩进实际上适合括号的预期嵌套，并且您已放回这些括号，则 CMq 不应更改任何内容。



## TODO 19.4 测试覆盖率

您可以通过加载 testcover 库并使用命令 Mx testcover-start RET file RET 来检测代码，对 Lisp 代码文件进行覆盖测试。  然后通过调用一次或多次来测试您的代码。  然后使用命令 Mx testcover-mark-all 在代码上显示彩色高亮显示覆盖不足的地方。  命令 Mx testcover-next-mark 将向前移动指向下一个突出显示的点。

通常，红色高亮表示表单从未被完全评估过；  棕色突出显示意味着它总是评估为相同的值（意味着几乎没有测试对结果所做的事情）。  但是，对于无法完成评估的表单（例如错误），将跳过红色突出显示。  对于预期总是评估为相同值的表单（例如 (setq x 14)），将跳过棕色突出显示。

对于困难的情况，您可以在代码中添加无操作宏来为测试覆盖率工具提供建议。

    Macro: 1value form ¶

评估表单并返回其值，但通知覆盖测试表单的值应始终相同。

    Macro: noreturn form ¶

评估表单，通知覆盖测试该表单永远不会返回。  如果它确实返回，你会得到一个运行时错误。

Edebug 还具有覆盖测试功能（请参阅覆盖测试）。  这些功能部分相互重复，将它们组合起来会更干净。



## TODO 19.5 剖析

如果您的程序运行正常，但速度不够快，并且您想让它运行得更快或更有效，那么首先要做的是分析您的代码，以便您知道它在哪里花费了大部分执行时间。  如果您发现某个特定功能占了执行时间的很大一部分，您可以开始寻找优化该部分的方法。

Emacs 对此有内置支持。  要开始分析，请键入 Mx profiler-start。  您可以选择定期采样 CPU 使用率 (cpu)、分配内存时 (memory)，或两者兼而有之。  然后运行您想要加速的代码。  之后，键入 Mx profiler-report 以显示您选择分析的每种类型（cpu 和内存）采样的 CPU 使用情况摘要缓冲区。  报告缓冲区的名称包括生成报告的时间，因此您可以稍后生成另一个报告，而不会删除以前的结果。  完成分析后，键入 Mx profiler-stop（与分析相关的开销很小，因此我们不建议将其保持活动状态，除非您实际运行要检查的代码）。

分析器报告缓冲区在每一行显示一个被调用的函数，前面是它使用了多少 CPU 资源，以绝对和百分比表示，自分析开始。  如果给定行的函数名称左侧有一个“+”符号，您可以通过键入 RET 扩展该行，以便查看更高级别函数调用的函数。  使用前缀参数 (Cu RET) 查看函数下方的整个调用树。  再次按 RET 将折叠回原始状态。

按 j 或 mouse-2 跳转到该点的函数定义。  按 d 查看函数的文档。  您可以使用 Cx Cw 将配置文件保存到文件中。  您可以使用 = 比较两个配置文件。

elp 库提供了一种替代方法，当您提前知道要分析哪些 Lisp 函数时，该方法很有用。  使用该库，您首先将 elp-function-list 设置为函数符号列表——这些是您要分析的函数。  然后键入 Mx elp-instrument-list RET nil RET 以安排分析这些功能。  运行要分析的代码后，调用 Mx elp-results 以显示当前结果。  有关更详细的说明，请参阅文件 elp.el。  这种方法仅限于分析用 Lisp 编写的函数，它不能分析 Emacs 原语。

您可以使用基准库来衡量评估单个 Emacs Lisp 表单所花费的时间。  在 benchmark.el 中查看函数 benchmark-call 以及 benchmark-run、benchmark-run-compiled、benchmark-progn 和 benchmark-call 宏。  您还可以交互地使用基准命令来计时表格。

要在其 C 代码级别分析 Emacs，您可以使用 configure 的 &#x2013;enable-profiling 选项构建它。  当 Emacs 退出时，它会生成一个文件 gmon.out，您可以使用 gprof 实用程序检查该文件。  此功能主要用于调试 Emacs。  它实际上停止了上述 Lisp 级别的 Mx profiler-&#x2026; 命令的工作。

