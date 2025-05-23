# 研发日志

## 2024.9.12
1. 已完成第一版的研发，但由于缺少装甲板，无法在实车上试验，但是在单装甲板上测试，在装甲板上的测试结果还行，项目暂时只能缓慢推进
2. 发现选板逻辑不正确，对于pnp处理的yaw值与实际车的yaw值关系不正确，需要更精确的yaw值计算方法

## 2024.9.26 17：23
1. 开始研究上交开源，对于yaw值计算的方法有了更多的想法，不过上交的代码庞大，计算复杂，仍在研究中
2. 思考yaw值计算方法
3. 需要一个同时保存所有装甲板的类来更好的管理整车运动

## 2024.9.27
1. 梳理清楚了如何进行yaw值投影计算，看懂了部分上交开源，学习到了三分法求极值的方法，开始进行代码转换

## 2024.9.28
1. 整理思路，写好了大部分代码，编译通过，但是还没有进行验证测试

## 2024.9.29
1. 测试代码，解决了n多问题，再一次对坐标系的变换进行了思考和修正，成功得到了一个投影yaw值
### 问题：
1. 仍有一定的跳动
2. 对于装甲板倾斜角要求高，因为是通过15度来计算的，所以如果装甲板倾斜角不是15度会导致一定的拟合失效或不准确
3. 似乎yaw值的0度角不正确，还需要进一步验证yaw值的计算准确度
### NEXT：
1. 继续yaw值计算的优化
2. 多装甲板处理设计
3. 弹道闭环的思考

## 2024.10.1
### 思考
1. 帧对帧处理，相机帧与发射帧考虑，对时间戳进行优化
2. 四装甲板描绘思考确定

## 2024.10.2
1. 开始编写弹道闭环代码，开发功能包closed_loop
### 发现的问题
1. 是否因为我们求相机坐标系到装甲板坐标系是pitch转动为yaw（而不是yaw为yaw），所以我们在tracker中也应该将pitch赋值给yaw，而不是yaw给yaw

## 2024.10.3-2024.10.8
1. 写完了绘制全车装甲板的代码，但测试能有一定的问题，对于处理完后的odom坐标还无法正确投影回像素坐标，估计原因为yaw值的计算方法和坐标系转换关系之间的不正确导致，正在测试检测

## 2024.10.9
1. 解决了处理完后的odom坐标还无法正确投影回像素坐标的问题-坐标变换进行错误，用错了tf2::doTransform导致，目前能够在弹道闭环中绘制出当前装甲板，但是对于全车装甲板的绘制还存在一定的问题，猜测可能原因是yaw正确，但是通过yaw值处理坐标时cos、sin的使用错误，导致从车中心变换回四个装甲板时出现了相应位置不正确的问题，正在测试检测
2. 解决完全车装甲板绘制后需要开展图片弹道模拟的测试，开始思考方案

## 2024.10.10
### 修改了多处有误计算
1. 测试结果：确定计算方法有误，应该为x - r*sin(i * CV_PI / 2 + yaw)，z + r*cos(i * CV_PI / 2 + yaw)，同理修改了tracker里面追踪器的算法，为装甲板点与车中心点的转换方法有误 -- 已修改
2. 处理的地方msg->c_to_a_pitch可能要个负值，因为pitch转动方向和用rotate函数的方向相反，先前在给camera_optical_frame到odom的变换时，pitch的值为- CV_PI - yaw, 那么这里应该把pitch取负值，测试结果：取负值 -- 已修改
3. 采用投影方法计算yaw时pitch计算方法有误，写成了- CV_PI / 180 * 2 - yaw（纯傻了，幸好发现了） --》 已修改为 - CV_PI - yaw
4. 对绘制全车装甲板的一些绘制问题进行了修改以及优化了显示

### 成功绘制全车装甲板 -- 2024.10.10
### NEXT
1. 弹道模拟的编写

## 2024.10.11
### 2024.10.11初步思考：
1. 统一采用odom坐标系进行弹丸坐标的计算 -- 因为odom坐标系是固定的，而其他坐标系随云台而发生变化
2. 将每一帧的图像保存下来，用于后续的闭环检测，主要存储时间戳信息，并定义一个初始时间，用于计算时间差，减少时间长度 // 时间ms单位
3. 使用自定义map队列，用于存储图像信息，当图像数量超过一定数量时，删除最早的图像信息，将新的图像信息插入队列，保存一定时间内的图像信息，减少内存占用，加快检索
4. 主要解算信息：装甲板在图像时间戳下的坐标（目标）| 发射点对于图像时间戳下打击的装甲板的坐标（起点），发射时间，发射速度，发射角度，飞行时间 -- 用于识别的弹丸的模拟弹道解算
5. 对于每一帧图片，制作模拟弹丸轨迹的可视化，定时间处理（timer处理）or其他时间处理方法？超过飞行时间的弹丸轨迹不再显示

#### 如何闭环？
1. 难点：只能获得弹丸在二维图像的像素坐标，而不能获得弹丸在三维空间的坐标，如何将二维坐标转换为三维坐标？ --》关键：弹丸小！！！
2. 比对弹丸在二维图像中的坐标和模拟弹道此时弹丸位置坐标的接近程度来判断符合哪一条模拟弹道？ --》 如果能准确匹配的话，就可以再通过模拟弹道和实际弹丸的误差来进行闭环检测

## 2024.10.12
1. 完成了弹道闭环模拟弹道的程序编写，需要进行测试可行性

## 2024.10.13
1. 粗劣的弹道闭环效果，因为有真实坐标转换的需要，所以具体测试和调试将在实车上进行
2. 进行弹道闭环第二项：真实弹道反馈 -- 需要识别小弹丸
3. 思考小弹丸识别方式

## 2024.10.14 - 2024.10.15
1. 编写小弹丸识别，运用传统视觉识别，但是效果不是很好，会有误识别，需要进一步检测来减少误识别的发生

## 2024.10.16 - 2024.10.18
1. 将小弹丸识别程序进行了一定的优化，减少了误识别率，但同时识别率降低，但需要进一步判断弹丸飞行过程中的识别率