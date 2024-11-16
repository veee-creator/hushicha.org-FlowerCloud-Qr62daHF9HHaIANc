
# 二游GAMELauncher启动器


## 1\.前言


* 许多二次元手游（原神，鸣潮，少女前线）的PC端启动器都是使用Qt做的，正好最近正在玩鸣潮，心血来潮，便仿鸣潮启动器，从头写一个。先下载一个官方版的PC启动器，找到图标，背景图等素材，然后对着界面写代码就行。
* 效果如下


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2375f6aee8244d79a4d945d1157fc2ee.png#pic_center)


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/d6770b4f50dd4363b15cdad9cc568d9f.png#pic_center)


## 2\. 划分模块


* 游戏启动器大致可以分为六部分


	+ 主体窗口
	+ 顶部标题栏
	+ 公告栏
	+ 轮播图
	+ 游戏下载模块
	+ 设置对话框
* 模块划分后，要做的事就很清晰了，对每一个模块，都新建一个带ui(方便布局)的类，然后根据各模块功能分别实现，最后组装在一起就行。


## 3\. 主体窗口


* 主体窗口是一个无边框窗口，然后有动态的背景图，有logo，版本号，版本标题。


	+ Qt中设置无边框窗口很简单，只要一行代码即可
	
	
	
	```
	  this->setWindowFlags(Qt::FramelessWindowHint | windowFlags());
	
	```
	
	这会导致窗口原本的移动事件和缩放事件无效，移动事件我们留在标题栏部分实现，缩放事件我们则不需要。
	+ 动态背景图，其实是通过定时切换图片实现，这时我们很容易想到使用定时器实现，到时间就就加载下一张图片。
	
	
	这样做会有一个问题，加载图片是需要时间的，这样做界面会有卡顿感。我们可以先把所有图片加载到内存，然后就不需要加载了，可以解决卡顿感。
	
	
	但是这又会导致另一个问题，图片很多，全部加载会占用很多内存空间，不够优雅。这里就需要用到线程，我们可以使用线程加载图片，然后通过信号把加载好的图片发送给主窗口就行绘制，这样既不卡顿也不占用很多内存。
	+ 新建一个加载图片的类(LoadImage)，继承QThread，实现run方法。
	
	
	
	```
	   while (!stop)
	   {
	        if (!imgNameList.empty())
	        {
	            QPixmap pix = QPixmap(imgNameList[curIndex]);
	            curIndex = (curIndex + 1) % imgNameList.size();
	            emit sendPixmap(pix);
	        }
	        QThread::msleep(fps);
	    }
	
	```
	
	我们可以设定帧数，让背景图实现指定帧率刷新。
	+ 这样我们就实现了背景图的切换
	
	
	![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a1a2ebaf2da346788f7036f0982c5710.png#pic_center)
	+ 然后我们把那些logo，版本号，标题，根据对应位置绘制上去就行了。logo 和 slogan都是图片来的。
	
	
	
	```
	 //绘制logo 和 slogan
	    p.drawImage(0,this->height()-slogan.height(),this->slogan);
	    p.drawImage(50,120,this->logo
	   //绘制版本号
	    QPen pen;
	    pen.setWidth(1);
	    pen.setColor(Qt::white);
	    p.setPen(pen);
	    QFont font("Arial", 12, QFont::Bold);
	    p.setFont(font);
	    p.drawText(10,this->height()-10,versionNumber);
	
	```
	+ 这样就得到了主体视觉图
	
	
	![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/637c0f5cb1c34403a25df9f2334e8efe.png#pic_center)
* 主体窗口还需要接收来自标题栏的移动，最小化，关闭的信号。



```
    connect(ui->topBar,&TopBar::miniumWindow,[this]()
    {
        this->showMinimized();
    });
    connect(ui->topBar,&TopBar::closeWindow,[this]()
    {
        this->close();
    });
    connect(ui->topBar,&TopBar::moveWindow,[this](QPoint pos)
    {
        this->move(pos+this->pos());
    });

```


## 4\. 顶部标题栏


* 标题栏用一个QWidget,然后把背景颜色设置成**rgba(0,0,0,80\)** 就可以实现透明的样式。其它的控件就是很常规的，里面有些按钮是有渐变的背景色和底部有白线，我们可以用一个QWidget加一个QPushButton作为一个组件实现，使用QWidget控件方便绘制白线。还有鼠标悬浮时显示的类似气泡的对话框，这个对话框需要自己实现。


	+ 气泡框可以通过把QWidget设置为无边框和透明窗口，然后里面绘制一个圆角矩形，然后再画一个三角形箭头即可。
	
	
	
	```
	setWindowFlags(Qt::FramelessWindowHint);
	setAttribute(Qt::WA_TranslucentBackground);
	
	void BubbleWidget::paintEvent(QPaintEvent*event)
	{
	
	    QPainter painter(this);
	    painter.setRenderHint(QPainter::Antialiasing);
	
	    // 设置背景颜色和边框
	    painter.setBrush(Qt::white);
	    painter.setPen(QPen(Qt::gray, 1));
	
	    // 创建圆角矩形路径
	    QPainterPath path;
	    QRectF rect = this->rect().adjusted(1, 10, -1,1); // 为箭头留出空间
	
	    path.addRoundedRect(rect, 6, 6);
	    painter.drawPath(path);
	    path.clear();
	    // 添加三角形箭头
	    int arrowWidth = 15;
	    int arrowHeight = 6;
	    QVectorpoints =
	    {
	        QPointF(rect.center().x() - arrowWidth / 2, rect.top()),
	        QPointF(rect.center().x() + arrowWidth / 2, rect.top()),
	        QPointF(rect.center().x(), rect.top() - arrowHeight)
	    };
	    path.addPolygon(QPolygonF(points));
	
	    // 绘制路径
	    painter.drawPath(path);
	
	```
	+ 然后我们可以根据需要往这个气泡框设置不同的Layout，就可以实现不同的布局效果了。
	
	
	![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b774059284d448a99167b2b0189ca38c.png#pic_center)
	+ 把其他控件放上就可以得到下面的标题栏，我们在这个类里面把移动，关闭，最小化的信号发给父窗口即可。
	![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/398339e695084b9c8687390851cf46d1.png#pic_center)
	+ 然后得到一个带标题栏的窗口。
	![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7d9231a4b6bd4d8faf31c5ac3f892654.png#pic_center)


## 5\. 公告栏


* 公告栏可以用QFrame和QStackedWidget组合实现，每条公告需要自定义一个QWiget来表示，处理好气泡框提示以及绘制左则的竖线。剩下就是对样式的设置，需要慢慢调一下。


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/ed81719b12ca4b238d78afd33eebe384.png#pic_center)


## 6\. 轮播图


* 轮播图使用QWiget和两个QPushButton实现，按钮固定在中间的左右两则，鼠标进入轮播图时显示。QWiget负责绘制轮播图片，图片切换是带一定动画效果的，不能直接切换图片。


	+ 轮播图上使用缓出动画效果，使得切换图片时更平滑。我们可以使用Qt的属性动画QPropertyAnimation，让图片位置属性**offset** 按缓和曲线进行变动，然后根据属性变化绘制当前图片和下一张图片即可。
	
	
	
	```
	  animation = new QPropertyAnimation(this, "offset");
	  animation->setStartValue(0.0);
	  animation->setEndValue(1.0);
	  animation->setDuration(400);
	  animation->setEasingCurve(QEasingCurve::OutCubic);   //缓出效果
	
	  void Carousel::paintEvent(QPaintEvent*e)
	  {
	       QPainter p(this);
	    if(!left)
	    {
	
	        p.drawImage(QRect(-width() * offset, 0, width(), height()), imgArr.at(curIndex).scaled(width(),height(),Qt::KeepAspectRatio,Qt::SmoothTransformation));
	        // 绘制下一张图片
	        p.drawImage(QRect(width() * (1 - offset), 0, width(), height()), imgArr.at(nextIndex).scaled(width(),height(),Qt::KeepAspectRatio,Qt::SmoothTransformation));
	    }
	    else
	    {
	        p.drawImage(QRect(width() * offset, 0, width(), height()), imgArr.at(curIndex).scaled(width(),height(),Qt::KeepAspectRatio,Qt::SmoothTransformation));
	        // 绘制下一张图片
	        p.drawImage(QRect(-width() * (1 - offset), 0, width(), height()), imgArr.at(nextIndex).scaled(width(),height(),Qt::KeepAspectRatio,Qt::SmoothTransformation));
	
	    }
	  }
	
	```
	+ 效果如下
	
	
	![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f08da7ea0fa74288ab86567a7119c7ca.png#pic_center)
* 图片进行缩放时要使用**Qt::SmoothTransformation** ，不然图片会很模糊。


## 7\. 下载模块


* 这个模块就比较简单，使用QWidget\+QStackedWidget，实现下载界面和进入游戏界面的切换。


	+ 有一些细节注意，QLineEdit要实现有个图标在最右侧可以使用**addAction函数**，添加一个图标。当QLineEdit的文字内容过长时，要让光标位于最开始位置，可以设置
	
	
	**setCursorPosition(0\)** 。还需要把QLineEdit自动获取鼠标焦点功能禁用，设置**setFocusPolicy(Qt::NoFocus)** 。
	+ ![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a94b75f5ed6e4804aef246a049f86cbd.png#pic_center)


## 8\. 设置


* 设置界面比较麻烦，里面的QCheckBox和QRadioButton的效果无法通过QSS实现，需要重写，里面的漏斗形图形需要比较多步骤去绘制。


	+ 重写QCheckBox
	
	
	
	```
	 void paintEvent(QPaintEvent *event) override
	    {
	        QCheckBox::paintEvent(event);
	        QPainter painter(this);
	
	
	        // 绘制复选框
	        QStyleOptionButton opt;
	        initStyleOption(&opt);
	
	        QRect checkBoxRect = style()->subElementRect(QStyle::SE_CheckBoxIndicator, &opt, this);
	        painter.setRenderHint(QPainter::Antialiasing);
	
	        if (isChecked())
	        {
	            // 绘制选中时的圆角背景
	            painter.setBrush(QColor("#BB9F5E"));  // 设置选中时的背景颜色
	            painter.setPen(Qt::NoPen);  // 去除边框
	        }
	        else
	        {
	            // 绘制未选中时的圆角边框
	            painter.setBrush(Qt::NoBrush);  // 不填充背景
	            painter.setPen(QPen(QColor("#8C8C8C"), 2));  // 使用灰色边框，线宽为2
	        }
	
	        // 绘制圆角矩形，圆角半径为3
	        painter.drawRoundedRect(checkBoxRect.adjusted(1, 1, -1, -1), 3, 3);
	
	        // 如果复选框被选中，绘制白色的勾
	        if (isChecked())
	        {
	            painter.setPen(QPen(Qt::white, 2));  // 设置勾的颜色为白色，线宽为2
	
	            // 使用 QPainterPath 绘制勾的形状
	            QPainterPath checkMarkPath;
	            checkMarkPath.moveTo(checkBoxRect.left() + checkBoxRect.width() * 0.3, checkBoxRect.center().y());
	            checkMarkPath.lineTo(checkBoxRect.center().x()-2, checkBoxRect.bottom() - checkBoxRect.height() * 0.3);
	            checkMarkPath.lineTo(checkBoxRect.right() - checkBoxRect.width() * 0.3, checkBoxRect.top() + checkBoxRect.height() * 0.35);
	            painter.drawPath(checkMarkPath);  // 绘制勾
	        }
	
	    }
	
	```
* 重写QRadioButton



```
  void paintEvent(QPaintEvent* event) override
    {

          QPainter painter(this);
          painter.setRenderHint(QPainter::Antialiasing);

          QStyleOptionButton option;
          initStyleOption(&option);
          painter.save();
  
  
  
          // 获取单选框的矩形区域
          QRect radioButtonRect = style()->subElementRect(QStyle::SE_RadioButtonIndicator, &option, this);
  
          // 增大单选框的尺寸
          int enlargedSize = 24;  // 自定义单选框的大小（增大后的大小）
          radioButtonRect.setWidth(enlargedSize);
          radioButtonRect.setHeight(enlargedSize);
  
  
  
          painter.setBrush(Qt::NoBrush);  // 不填充背景
          painter.setPen(QPen(QColor("#8C8C8C"), 2));  // 使用灰色边框，线宽为2
  
          // 绘制增大的圆形的单选框
          QRect circleRect = radioButtonRect.adjusted(2, 2, -2, -2); // 调整绘制圆形的位置
          painter.drawEllipse(circleRect);
  
          // 如果当前单选框被选中，则填充中心
          if (isChecked()) 
          {   
  
              painter.setPen(QPen(QColor("#BB9F5E"), 2));
              painter.drawEllipse(circleRect);
  
  
              painter.setBrush(QColor("#BB9F5E"));   // 设置选中时的填充颜色为白色
              painter.drawEllipse(circleRect.adjusted(5,5, -5, -5));  // 绘制小圆圈，表示选中
          }
  
          // 绘制文本，确保文本位置对齐
          QRect textRect = option.rect;
  
          // 将文本左移，使其与增大的单选框右边对齐
          textRect.setLeft(radioButtonRect.right() + 5);  // 将文本移到单选框右侧
  
          // 使文本垂直居中
          textRect.moveTop(radioButtonRect.top() + (radioButtonRect.height() - textRect.height()) / 2);
          painter.restore();
  
          // 使用默认的文本颜色（由样式表和控件状态决定）
          style()->drawItemText(&painter, textRect, Qt::AlignVCenter, option.palette, isEnabled(), option.text);
      }

```
* 绘制设置框的线条和图形。



```
void Setting::paintEvent(QPaintEvent*event)
{   
      QPainter p(this);
      QPen pen;
      QPainterPath path;
      pen.setColor(QColor("#CFCFCF"));//CFCFCF
      pen.setWidth(2);
      p.setPen(pen);
  
      //画顶部线条
      int x = ui->labelSetting->pos().x();
      int y = ui->labelSetting->pos().y()+ui->labelSetting->height()+10;
      p.drawLine(x,y,x+this->width()-40,y);
  
  
  
  
      //画圆弧
      int aw = 20;
      int endx = x+this->width()-35;
      int endy = y;
  
      int cx = endx-aw;
      int cy = endy-aw;
  
      int tx = cx-aw;
      int ty = cy-aw;
  
      //int bx = cx+aw;
      int by = cy+aw;
  
  
      int startAngle = 270;
      int spanAngle = 80;
      double rr = aw;
  
      int cx1 = tx+aw;
      int cy1 = ty+aw;
      double ex1 = cx1 + rr * cos((startAngle + spanAngle) * 3.14 / 180);
      double ey1 = cy1 - rr * sin((startAngle + spanAngle) * 3.14 / 180);
  
      startAngle = 90;
      spanAngle  = -80;
      int cx2 = tx+aw;
      int cy2 = by+aw;
      double ex2 = cx2 + rr * cos((startAngle + spanAngle) * 3.14 / 180);
      double ey2 = cy2 - rr * sin((startAngle + spanAngle) * 3.14 / 180);
  
  
  
      p.setBrush(QColor("#333333"));
      p.setPen(Qt::white);
      path.moveTo(cx,by);
      path.lineTo(ex1,ey1);
      path.lineTo(ex2,ey2);
      path.lineTo(cx,by);
      p.drawPath(path);
  
  
  
      path.clear();
      path.moveTo(cx,by);
      QRect r (tx,ty,aw*2,aw*2);  //x,y,width,height
      QRect r2(tx,by,aw*2,aw*2);
  
      pen.setWidth(1);
      p.setPen(pen);
      p.setRenderHint(QPainter::Antialiasing);
      path.arcTo(r,270,80);
      path.moveTo(cx,by);
      path.arcTo(r2,90,-80);
      p.fillPath(path,Qt::white);
  
  
  
      //画顶部线条
      pen.setColor(QColor("#464646"));
      pen.setWidth(2);
      p.setPen(pen);
      p.drawLine(x+this->width()-35,0,x+this->width()-35,this->height());
  
  
      //画右边线条
      int s1x = ui->btnCancel->pos().x();
      int s1y = ui->btnCancel->pos().y()-20;
      int s2x = ui->btnOk->pos().x()+ui->btnOk->width();
  
      pen.setColor(QColor("#CBCBCB"));
      pen.setWidthF(1.5);
      p.setPen(pen);
      p.drawLine(s1x,s1y,s2x,s1y);
  
  
      //画右下角三角形
      QPoint p1(x+this->width()-35,this->height()-10);
      QPoint p2(x+this->width()-35,this->height());
      QPoint p3(x+this->width()-45,this->height());
  
  
      QPolygon cons;
      cons<setPen(Qt::black);
      p.drawPolygon(cons);
  }
  

```
* 最终效果图就是这样。


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b0b322fe95924f988e2d77f1b6eb490b.png#pic_center)


## 9\. 其它


* 除了界面之外，我们编写一下各控件对应事件就可以了，比如打开链接，跳转到网站。
* 完整的源码在放在github里面了：[GameLauncher](https://github.com)
* 有玩鸣潮的可以加个好友：**ID:100073367**


 本博客参考[westworld加速](https://tianchuang88.com)。转载请注明出处！
