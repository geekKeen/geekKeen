# 数据结构--排序算法

---


## 排序算法
本文贴出了关于排序算法中的几个经典算法，重点了解快排和堆排；
本文排序的测试用例为：`[49, 38, 65, 97, 76, 13, 27, 49(a)]`, 其中的 `49(a)` 表示数组中的重复元素，说明算法是否是稳定的排序算法。
本文贴出的都是代码，详尽的原理和功能在注释中做出了简单说明；

## 插入排序

### 直接插入排序

```C
void straight_insert(vector<int> nums)
{
    /*
    *  功能：直接插入排序
    *  原理：假定数组的前面部分已经有序，检测当前索引插入的位置 j;
    *        插入位置后的元素一致后移
    *  说明： O(n^2) Time and O(1) Space, 稳定的排序算法,
    *        为了不影响其他排序算法，参数传递时未添加引用
    */
    int privot = nums[0];
    for (int i = 1; i < nums.size(); ++i)
    {
        if (nums[i] < nums[i - 1])
        {
            privot = nums[i]; // 设置哨兵，作为临时空间
            // 从右往左搜索， 找到第一个小于 privot 元素的位置
            int j = i - 1;
            while (j >= 0 && privot < nums[j])
            {
                nums[j + 1] = nums[j];
                --j;
            }
            nums[j + 1] = privot; // Insert
        }
    }
    cout << "Straight Insert Result: ";
    print(nums);
}
```

---

### Shell 排序

希尔排序的过程

```c
void shell_sort(vector<int> nums)
{
    /*
    * 功能：Shell 排序
    * 原理：Shell 排序是基于直接插入排序的算法，
    *       将数组元素按增量划分为一组，
    *       如 delta = 4，则 位置为[0,4,8,...]为一组，位置为[1,5,9，...]的元素作为一组
    *       组内元素做直接插入排序；
    *       增量设置为[n/2, n/4, n/8, ... ,1]
    * 说明：Shell 排序的时间复杂度为小于 O(N^2),由于增量的选择，Shell 排序是不稳定排序
    *       为了不影响其他排序算法，参数传递时未添加引用
    */
    int delta = nums.size() / 2;
    while (delta >= 1)
    {
        _shell_insert(nums, delta);
        delta /= 2;
    }
    cout << "Shell Insert Result: ";
    print(nums);
}
```

基于直接插入排排序的增量排序过程

```c
void _shell_insert(vector<int> &nums, int delta)
{
    /*
    * 功能：Shell 排序的基于增量的直接插入排序
    * 原理：每隔增量的元素作为一组，进行直接插入排序
    */
    int privot = nums[0];
    for (int i = delta; i < nums.size(); ++i)
    {
        if (nums[i] < nums[i - delta])
        {
            privot = nums[i];
            int j = i - delta;
            while (j >= 0 && nums[i] < nums[j])
            {
                nums[j + delta] = nums[j];
                j -= delta;
            }
            nums[j + delta] = privot;
        }
    }
}
```

---

## 选择排序

### 简单选择排序

```c
void simple_select_sort(vector<int> nums)
{
    /*
    * 功能：简单选择排序
    * 原理：数组的左面表示有序部分，每一趟循环完成，确定其余元素的最大（最小）值，并纳入有序部分
    * 说明: Time O(N^2), 因为存在跨元素交换，所以是不稳定的排序算法
    *       为了不影响其他排序算法，参数传递未添加引用
    */
    int position = 0;
    for (int i = 0; i < nums.size() - 1; ++i)
    {
        position = i;
        for (int j = i+1; j < nums.size(); ++j)
        {
            if (nums[position] > nums[j])
                position = j;
        }
        swap(nums[i], nums[position]);
    }
    cout << "Simple Select Result: ";
    print(nums);
}
```

---

###堆排序

堆排序的主过程

```c
void heaq_sort(vector<int> nums)
{
    /*
    * 功能：堆排序算法
    * 原理：利用数组可以模拟完全二叉树的原理，使得父节点与子节点存在对应联系
    *       堆排序是对简单选择排序的优化，认为数组的右边为有序序列
    *       通过循环以下过程，得到有序数组
    *       1.建立大根（小根）堆，找出最大（最小）值，优化了简单排序的选择过程
    *       2.将得到的最值与无效序列的最后一个值互换
    *       1,2 过程循环，即可得到有序数组
    * 说明：由于需要建立根堆这个特性，使得堆排序用于筛选数组最大（最小）n 个元素
    *       Time O(N*logN),与简单选择相同，存在跨元素交换，所以是不稳定排序
    *       为了不影响其他排序算法，参数传递未添加引用
    *
    */
    for (int i = 0; i < nums.size(); ++i)
    {
        _build_heaq(nums, nums.size() - i);
        swap(nums[0], nums[nums.size()-i-1]);
    }
    cout << "Heaq Sort Result: ";
    print(nums);
}
```

建立堆栈的过程

```c
void _build_heaq(vector<int> &nums, int length)
{
    /*
    * 功能：建立根堆 
    * 原理：建堆的过程是从下往上的，首先需要找到第一个存在子节点的节点
    *       每找到一个节点，则已此结点为根的树，需要调整元素，完成建堆过程
    */
    int i = (nums.size() - 1) / 2;
    while (i >= 0)
    {
        _heaq_adjust(nums, i, length);
        --i;
    }
}
```

堆栈节点的调整实现， 使得以该节点为根的树符合根堆特征

```c
void _heaq_adjust(vector<int> &nums, int curr,int length)
{
    /*
    * 功能：堆调整，使得以该节点为根的树，都符合大根堆特征
    * 原理；比较当前节点与其较大子节点的大小，如果大于，不用调整
    *       如果小于，则交换，并将该子节点设为需要调整的节点
    */
    int child;
    while (curr < length)
    {
        child = 2 * curr + 1;
        if (child < length)
        {
            if (child + 1 < length && nums[child] < nums[child + 1])
                ++child;
            if (nums[curr] < nums[child])
            {
                swap(nums[curr], nums[child]);
                curr = child;
            }
            else
                break;
        }
        else
            break;
    }
}

```

---

## 交换排序

###冒泡排序

```c
void bubble_sort(vector<int> nums)
{
    /*
    * 功能：冒泡排序算法
    * 原理：相邻元素之间两两比较，交换；当每次比较整个数组时，最大（最小元素）被交换到指定位置
    * 说明：冒泡排序的时间复杂度为 O(N^2), 因为是相邻元素比较，所以是稳定的排序算法
    */
    for (int i = 0; i < nums.size()-1; ++i)
    {//冒泡排序的次数
        for (int j = 1; j < nums.size() - i; ++j)
        {//将除已确定的元素进行冒泡排序
            if (nums[j] < nums[j - 1])
            {
                swap(nums[j], nums[j-1]);
            }
        }
    }
    cout << "Bubble Sort Result: ";
    print(nums);
}
```

---

###快速排序

快排的主过程
```c
void quick_sort(vector<int> &nums, int low, int high) 
{
    /*
    * 功能：快排算法
    * 原理：快排运用的是分治法的原理
    *       1. 将数组中的元素划分为大于等于 key 和小于 key 的部分
    *       2. key 左右两边再进行快速排序，
    *       3. 经过多次的快排使得各个部分有序，则整体有序
    * 说明：Time O(NlogN), 因为选择的 key 都是第一个或者最后一个元素，所以是不稳定的算法
    */
    if (low < high)
    {
        int key_position = _quick_sort_swap(nums, low, high);
        quick_sort(nums, 0, key_position - 1);
        quick_sort(nums, key_position + 1, high);
    }
}
```

通过交换，将数组划分为大于等于 Key 和小于等于 key 的过程

```c
int _quick_sort_swap(vector<int> &nums, int low, int high)
{
    /*
    * 功能；利用交换的方法，将数组划分为大于等于key和小于key两个部分，并返回key位置
    * 原理：利用 Two pointers 方法，第一个元素或者最后一个元素作为 key
    */
    int key = nums[low];
    while (low < high)
    {
        while (low < high && nums[high] >= key)
            --high;
        swap(nums[high], nums[low]);
        while (low < high && nums[low] < key)
            ++low;
        swap(nums[high], nums[low]);
    }
    return low;
}
```

---

##归并排序

归并排序的主过程

```c
void merge_sort(vector<int> nums)
{
    /*
    * 功能：归并排序
    * 原理：将有序的序列合并，将数组划分为多个有序的序列，然后合并
    *       1. 只有一个元素的序列是有序的；
    *       2. 将一个元素的序列合并，则数组会被划分为多个元素数目为 2 的序列；
    *       3. 直到元素数目为数组大小的序列，该序列中数组有序；
    * 说明：Time O(NlogN)， Space O（N）；都是相邻的元素的比较,所以是稳定的排序算法
    */
    vector<int> temp(nums.begin(), nums.end());
    int group_len = 1;
    int double_group_len;
    while (group_len <= nums.size())
    {
        int i = 0;
        double_group_len = 2 * group_len;
        while (i + double_group_len <= nums.size())
        {
            _merge(nums, temp, i, i + group_len - 1, i + double_group_len - 1);
            i += double_group_len;
        }
        if (i + group_len <= nums.size())
        {
            _merge(nums, temp,i, i+group_len-1, nums.size() - 1);
        }

        nums = temp;
        group_len = double_group_len;
    }
    cout << "Merge Sort Result: ";
    print(nums);  
}
```

合并两个有序序列的过程

```c
void _merge(vector<int> &nums, vector<int> &temp, int first_start, int first_end, int second_end)
{
    /*
    * 功能： 将两个有序的序列合并
    */
    int second_start = first_end + 1;
    int index = first_start;
    int i, j;
    for (i = first_start, j = second_start; i <= first_end && j <=second_end;)
    {
        if (nums[i] < nums[j])
            temp[index++] = nums[i++];
        else
            temp[index++] = nums[j++];
    }
    while (i <= first_end) temp[index++] = nums[i++];
    while (j <= second_end) temp[index++] = nums[j++];
}
```





