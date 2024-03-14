## public class MonoBehavior

调用A* ：
Start() -> FindPath();
各个函数作用：
```C#
Start();
//创建栅格地图，调用A* 算法找到路径，生成路径中的各个目标点，并在调试器中输出找到的这些点的信息

CreateGrid();
	//栅格地图是一个由0/1充满的二维数组，是否有障碍物由isObstacle判断与赋值
	//其中x、y是地图坐标
	//isObstacle是从pbstacleLayer中获取信息的
	grid[x,y] = new Node(isObstacle, worldPoint);

FindPath();
	//A*算法具体流程，可以结合原理看，但看不懂感觉也可以用，没啥问题
RetracePath();
	//缺少里面的关键变量，看不太懂，但好像是每在路径上走到一个新的点，就把这个带你加入到之前的路径里？
GetNeighbours();
	//看起来是获取周围点的地图信息，从grid[checkX, checkY]中读到了一些数据
GetDistance();
	//计算距离，得到的是A与B点之间距离的平方值
NodeFromWorldPoint();
	//看不太懂，似乎是从世界坐标系中创建了一个节点？
```
Start内函数加注释

```C#
    void Start()

    {

        CreateGrid();

        //A*算法规划轨迹

        Vector3 startPos = new Vector3(0, 0, 0); // 起始位置

        Vector3 targetPos = new Vector3(10, 10, 0); // 目标位置

        List<Node> path = FindPath(startPos, targetPos); // 使用A*算法找到路径

        // 在调试器中输出路径节点信息

        if (path != null)

        {

            foreach (Node node in path)

            {

                Debug.Log("Path node: " + node.worldPosition);

            }

        }

    }

```

## public class Node

用来存储路径中每一个目标点的信息
包括目前这个点的坐标，世界坐标系中的位置，是否是障碍物，移动成本等等