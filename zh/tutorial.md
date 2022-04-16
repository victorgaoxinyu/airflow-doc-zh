# 教程

> 贡献者：[@ImPerat0R\_](https://github.com/tssujt)、[@ThinkingChen](https://github.com/cdmikechen)、[@Ray](https://github.com/echo-ray)、[@zhongjiajie](https://github.com/zhongjiajie)

本教程将向您介绍一些 Airflow 的基本概念、对象以及它们在编写第一个 pipline（管道）时的用法。

## 定义 Pipeline（管道）的例子

以下是定义一个基本 pipline（管道）的示例。如果这看起来很复杂，请不要担心，下面将逐行说明。

```py
"""
Airflow 教程代码位于:
https://github.com/apache/airflow/blob/master/airflow/example_dags/tutorial.py
"""
from datetime import datetime, timedelta
from textwrap import dedent

# DAG 对象；我们需要用这个来实例化一个DAG
from airflow import DAG
# Operators；我们需要这个来执行操作！
from airflow.operators.bash_operator import BashOperator

with DAG(
    'tutorial',
    # 这些参数会被传入每一个operator中
    # 你可以在operator初始化的时候把每一个任务的参数进行覆写
    default_args={
        'depends_on_past': False,
        'email': ['airflow@example.com'],
        'email_on_failure': False,
        'email_on_retry': False,
        'retries': 1,
        'retry_delay': timedelta(minutes=5),
        # 'queue': 'bash_queue',
        # 'pool': 'backfill',
        # 'priority_weight': 10,
        # 'end_date': datetime(2016, 1, 1),
        # 'wait_for_downstream': False,
        # 'sla': timedelta(hours=2),
        # 'execution_timeout': timedelta(seconds=300),
        # 'on_failure_callback': some_function,
        # 'on_success_callback': some_other_function,
        # 'on_retry_callback': another_function,
        # 'sla_miss_callback': yet_another_function,
        # 'trigger_rule': 'all_success'
    },
    description='A simple tutorial DAG',
    schedule_interval=timedelta(days=1),
    start_date=datetime(2021, 1, 1),
    catchup=False,
    tags=['example'],
) as dag:

    # t1、t2 和 t3 是通过实例化 Operators 创建的任务示例

    t1 = BashOperator(
        task_id='print_date',
        bash_command='date',
    )

    t2 = BashOperator(
        task_id='sleep',
        depends_on_past=False,
        bash_command='sleep 5',
        retries=3,
    )
    t1.doc_md = dedent(
        """\
    #### Task Documentation
    You can document your task using the attributes `doc_md` (markdown),
    `doc` (plain text), `doc_rst`, `doc_json`, `doc_yaml` which gets
    rendered in the UI's Task Instance Details page.
    ![img](http://montcs.bloomu.edu/~bobmon/Semesters/2012-01/491/import%20soul.png)

    """
    )

    dag.doc_md = __doc__  # providing that you have a docstring at the beginning of the DAG
    dag.doc_md = """
    This is a documentation placed anywhere
    """  # otherwise, type it like this
    templated_command = dedent(
        """
    {% for i in range(5) %}
        echo "{{ ds }}"
        echo "{{ macros.ds_add(ds, 7)}}"
    {% endfor %}
    """
    )

    t3 = BashOperator(
        task_id='templated',
        depends_on_past=False,
        bash_command=templated_command,
    )

    t1 >> [t2, t3]
```

## 这是一个 DAG 定义文件

有一件事需要考虑(一开始可能不是很直观)，这个 Airflow 的 Python 脚本实际上只是一个将 DAG 的结构指定为代码的配置文件。此处定义的实际任务将在与此脚本定义的不同上下文中运行。不同的任务在不同的时间点运行在不同的 worker（工作节点）上，这意味着该脚本不能在任务之间交叉通信。请注意，为此，我们有一个名为`XCom`的更高级功能。

人们有时会将 DAG 定义文件视为可以进行实际数据处理的地方 - 但事实并非如此！该脚本的目的是定义 DAG 对象。它需要快速评估（秒，而不是几分钟），因为 scheduler（调度器）将定期执行它以反映更改（如果有的话）。

## 导入模块

一个 Airflow 的 pipeline 就是一个 Python 脚本，这个脚本的作用是为了定义 Airflow 的 DAG 对象。让我们首先导入我们需要的库。

```py
from datetime import datetime, timedelta
from textwrap import dedent

# DAG 对象; 我们将需要它来实例化一个 DAG
from airflow import DAG

# Operators; 我们需要利用这个对象去执行流程!
from airflow.operators.bash import BashOperator
```
参阅模块管理([Modules Management](??))来具体了解Python和Airflow是如何管理模块的。

## 默认参数

我们即将创建一个 DAG 和一些任务，我们可以选择显式地将一组参数传递给每个任务的构造函数（这可能变得多余），或者（最好地）我们可以定义一个默认参数的字典，这样我们可以在创建任务时使用它。

```py
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2015, 6, 1),
    'email': ['airflow@example.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
    # 'queue': 'bash_queue',
    # 'pool': 'backfill',
    # 'priority_weight': 10,
    # 'end_date': datetime(2016, 1, 1),
}

```

有关 BaseOperator 参数及其功能的更多信息，请参阅[airflow.models.BaseOperator](zh/31?id=baseoperator)文档。

另外，请注意，您可以轻松定义可用于不同目的的不同参数集。一个典型的例子是在生产和开发环境之间进行不同的设置。

## 实例化一个 DAG

我们需要一个 DAG 对象来嵌入我们的任务。这里我们传递一个定义为`dag_id`的字符串，把它用作 DAG 的唯一标识符。我们还传递我们刚刚定义的默认参数字典，同时也为 DAG 定义`schedule_interval`，设置调度间隔为每天一次。

```py
dag = DAG(
    'tutorial', default_args=default_args, schedule_interval=timedelta(days=1))

```

## （Task）任务

在实例化 operator（执行器）时会生成任务。从一个 operator（执行器）实例化出来的对象的过程，被称为一个构造方法。第一个参数`task_id`充当任务的唯一标识符。

```py
t1 = BashOperator(
    task_id='print_date',
    bash_command='date',
    dag=dag)

t2 = BashOperator(
    task_id='sleep',
    bash_command='sleep 5',
    retries=3,
    dag=dag)

```

注意到我们传递了一个 BaseOperator 特有的参数(`bash_command`)和所有的 operator 构造函数中都会有的一个参数(`retries`)。这比为每个构造函数传递所有的参数要简单很多。另请注意，在第二个任务中，我们使用`3`覆盖了默认的`retries`参数值。

任务参数的优先规则如下：

1. 明确传递参数
2. `default_args`字典中存在的值
3. operator 的默认值（如果存在）

任务必须包含或继承参数`task_id`和`owner`，否则 Airflow 将出现异常。

## 使用 Jinja 作为模版

Airflow 充分利用了[Jinja Templating](http://jinja.pocoo.org/docs/dev/)的强大功能，并为 pipline（管道）的作者提供了一组内置参数和 macros（宏）。Airflow 还为 pipline（管道）作者提供了自定义参数，macros（宏）和 templates（模板）的能力。

本教程几乎没有涉及在 Airflow 中使用模板进行操作的工作领域，但本节的目的是让您知道此功能的存在，让您熟悉`{{ }}`双花括号的用途，并指出最常见的模板变量： `{{ ds }}` （今天的“日期戳”）。

```py
templated_command = """
    { % f or i in range(5) %}
        echo "{{ ds }}"
        echo "{{ macros.ds_add(ds, 7) }}"
        echo "{{ params.my_param }}"
    { % e ndfor %}
"""

t3 = BashOperator(
    task_id='templated',
    bash_command=templated_command,
    params={'my_param': 'Parameter I passed in'},
    dag=dag)

```

请注意，`templated_command`包含`{% %}`块中的代码逻辑，引用参数如`{{ ds }}`，调用函数方式如`{{ macros.ds_add(ds, 7)}}`，引用用户定义的参数如`{{ params.my_param }}`。

在`BaseOperator`中的`params`hook 允许您将参数或对象的字典传递给您的模板。请花一些时间去了解`my_param`这个参数是如何在模板中被使用的。

文件也可以当做`bash_command`的参数进行传递，例如`bash_command='templated_command.sh'`，不过这个文件的位置要在 pipeline（管道）文件的目录内（在本例中为`tutorial.py`）。这可能是出于多种原因，比如将脚本的逻辑和 pipeline 代码分隔开，允许在使用不同语言编写的文件中进行正确的代码突出显示，以及灵活地构建 pipeline（管道）。还可以定义您的`template_searchpath`，以指向 DAG 构造函数调用中的任何文件夹位置。

使用同样的 DAG 构造函数调用，可以使用`user_defined_macros`来定义您自己的变量。例如，将`dict(foo='bar')`传递给此参数允许您在模板中使用`{{ foo }}` 。此外，允许您指定`user_defined_filters`来注册自己的过滤器。例如，将`dict(hello=lambda name: 'Hello %s' % name)`传递给此参数可以允许您在你的模板中使用`{{ 'world' | hello }}`。有关自定义过滤器的更多信息，请查看[Jinja 文档](http://jinja.pocoo.org/docs/dev/api/#writing-filters)

有关可以在模板中引用的变量和宏的更多信息，请务必阅读[宏](zh/code.md)部分

## 设置依赖关系

我们有三个不相互依赖任务，分别是`t1`，`t2`，`t3`。以下是一些可以定义它们之间依赖关系的方法：

```py
t1.set_downstream(t2)

# 这意味着 t2 会在 t1 成功执行之后才会执行
# 与下面这种写法相等
t2.set_upstream(t1)

# 位移运算符也可用于链式运算
# 用于链式关系 和上面达到一样的效果
t1 >> t2

# 位移运算符用于上游关系中
t2 << t1


# 使用位移运算符能够链接
# 多个依赖关系变得简洁
t1 >> t2 >> t3

# 任务列表也可以设置为依赖项。
# 下面的这些操作都具有相同的效果:
t1.set_downstream([t2, t3])
t1 >> [t2, t3]
[t2, t3] << t1
```

请注意，在执行脚本时，在 DAG 中如果存在循环或多次引用依赖项时，Airflow 会引发异常。

## 回顾

到此，我们有了一个非常基本的 DAG。此时，您的代码应如下所示：

```py
 """
Airflow 教程代码位于:
https://github.com/apache/airflow/blob/master/airflow/example_dags/tutorial.py
"""
from airflow import DAG
from airflow.operators.bash_operator import BashOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2015, 6, 1),
    'email': ['airflow@example.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
    # 'queue': 'bash_queue',
    # 'pool': 'backfill',
    # 'priority_weight': 10,
    # 'end_date': datetime(2016, 1, 1),
}

dag = DAG(
    'tutorial', default_args=default_args, schedule_interval=timedelta(days=1))

# t1, t2 and t3 are examples of tasks created by instantiating operators
t1 = BashOperator(
    task_id='print_date',
    bash_command='date',
    dag=dag)

t2 = BashOperator(
    task_id='sleep',
    bash_command='sleep 5',
    retries=3,
    dag=dag)

templated_command = """
    { % f or i in range(5) %}
    echo "{{ ds }}"
    echo "{{ macros.ds_add(ds, 7)}}"
    echo "{{ params.my_param }}"
    { % e ndfor %}
"""

t3 = BashOperator(
    task_id='templated',
    bash_command=templated_command,
    params={'my_param': 'Parameter I passed in'},
    dag=dag)

t2.set_upstream(t1)
t3.set_upstream(t1)
```

## 测试

### 运行脚本

是时候进行一些测试了。首先让我们确保 pipeline（管道）能够被解析。让我们保证已经将前面的几个步骤的代码保存在`tutorial.py`文件中，并将文件放置在`airflow.cfg`设置的 DAGs 文件夹中。DAGs 的默认位置是`~/airflow/dags`。

```bash
python ~/airflow/dags/tutorial.py
```

如果这个脚本没有报错，那就证明您的代码和您的 Airflow 环境没有特别大的问题。

### 命令行元数据验证

让我们运行一些命令来进一步验证这个脚本。

```bash
# 打印出所有正在活跃状态的 DAGs
airflow list_dags

# 打印出 'tutorial' DAG 中所有的任务
airflow list_tasks tutorial

# 打印出 'tutorial' DAG 的任务层次结构
airflow list_tasks tutorial --tree
```

### 测试实例

让我们通过在特定日期运行实际任务实例来进行测试。通过`execution_date`这个上下文指定日期，它会模拟 scheduler 在特定的 日期 + 时间 运行您的任务或者 dag：

```bash
# 命令样式: command subcommand dag_id task_id date

# 测试 print_date
airflow test tutorial print_date 2015-06-01

# 测试 sleep
airflow test tutorial sleep 2015-06-01
```

现在还记得我们早些时候利用模板都做了什么？让我们通过执行这个命令看看模板会被渲染成什么样子：

```bash
 # 测试模版渲染
airflow test tutorial templated 2015-06-01
```

用过运行 bash 命令，应该会显示详细的事件日志并打印结果。

请注意，`airflow test`命令在本地运行任务实例时，会将其日志输出到 stdout（在屏幕上），不会受依赖项影响，并且不向数据库传达状态（运行，成功，失败，...）。它只允许测试单个任务实例。

### Backfill（回填）

一切看起来都运行良好，所以此时让我们运行 backfill（回填）。`backfill`将尊重您的依赖关系，将日志发送到文件并与数据库通信以记录状态。如果您启动了一个 web 服务，您可以跟踪它的进度。`airflow webserver`将启动 Web 服务器，如果您有兴趣在 backfill（回填）过程中直观地跟踪进度。

请注意，如果使用`depends_on_past=True`，则单个任务实例的执行将取决于前面任务实例是否成功，除了以 start_date 作为开始时间的实例（即第一个运行的 DAG 实例），他的依赖性会被忽略。

此上下文中的日期范围是`start_date`和可选的`end_date`，它们用于使用此 dag 中的任务实例填充运行计划。

```bash
# 可选，在后台以 debug 模式运行 web 服务器
# airflow webserver --debug &

# 在时间范围内回填执行任务
airflow backfill tutorial -s 2015-06-01 -e 2015-06-07
```

## 接下来做什么

就如上面这样，您已经编写，测试并 backfill（回填）了您的第一个 Airflow 的 pipeline（管道）。将您的代码合并到一个有 scheduler（调度管理器）的代码库中，这样可以启动任务并在每天执行它。

以下是您可能想要做的一些事情：

* 深入了解用户界面 - 点击所有内容！
* 继续阅读文档！ 特别是以下部分：
  * 命令行界面
  * Operators（运营商）
  * Macros（宏）
* 写下你的第一个 pipline（管道）！
