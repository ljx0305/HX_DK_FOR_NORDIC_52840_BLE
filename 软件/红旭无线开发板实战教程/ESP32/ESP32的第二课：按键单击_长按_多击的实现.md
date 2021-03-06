## 前言
> 首先,给各位老铁置于最高的歉意,让各位老表等教程等得太久了(当然也有可能完全是我自作多情这么想),甚至有人说"红旭无线开发团队是不是跑路了?:joy:".
其实,完全没有这么回事,你们只要记住 **"芯在红旭就在,红旭在教程就不会断"**,请大家马上把这个加粗的字圈起来,以后"考试"要考.之所以会出现这样的情况,我这里简单地说下吧,人终究是感情的动物,这段时间因为遇到了一些锁碎的事要分身去处理,所以把教程耽搁了.如今,小编已经重新回到战斗岗位.不废话了,就跟习大大说地那样 **"撸起袖子加油干"**.

## 背景
一说到按键,我想不管是新鸟还是老鸟在嵌入式的职业生涯中必定会项目上有用到按键吧.在网络上也有很多大佬分享他们的按键心得体会.基本上的情况如下所示:

- 几个按键都是连续在一起的,即GPIO1,2,3,4,5在硬件上是连接在一起的,加上定时器轮询判断是哪个按键按下;

    **疑问:** 哪里有这么巧刚好按键的连接顺序都是GPIO口连着的?

- 链表+定时器的方法,定时轮询哪个按键按下并对相对应的按键动作做出处理；

    **疑问:** 这种写法的确很巧妙,但是如果要求的按键击数多的话,程序会不会很长很复杂?

- GPIO中断＋定时器的方法，GPIO口中断触发定时器,从而实现按键的逻辑判断

    **疑问:**
其实这个方向是对的,但是网络上分享的内容还是有所欠缺;
   - 如果我有多个按键需要单击/长按/多击时呢? 
   
以上这几种按键处理方法均是网络上能查找得到的.虽然大部分都可以用,但是在我看来还并不是那么的完美.总是让人感觉缺点什么,比如:
- 如果产品有低功耗要求呢?一个定时器频繁轮询是否有按键按下,这样真的好么?
- 如果产品存在有多个按键,并且每个按键均有单击/长按/多击且时长都不同的需求,那应怎么办呢?

那么针对以上的这些情况,那么有没有满足以上方法优点的同时把缺点也补上呢?答案很明显,当然是有的.随我抛砖引玉慢慢道来.

## 原理
首先,既然要判断按键是否是短按/长按/多击,那么定时器是不可以避免的,这是必需品绝对不可少.但是定时轮询这种方式是肯定不会采取的.那么最终需要哪些东西呢?如果全是文字的话,未免太过于枯燥难懂,下面我将以一张脑图展示这些内容:

![按键所需要的条件](https://raw.githubusercontent.com/xiaolongba/picture/master/%E5%8D%95%E5%87%BB_%E9%95%BF%E6%8C%89_%E5%A4%9A%E5%87%BB.png)
### GPIO中断
按键对应的GPIO口设置为双边沿触发，而不是通过定时器轮询判断.因为在小编看来定时器轮询效率太低,大部分时间按键都是处于未触发状态的.同时这样的做法对一些要低功耗处理的场合并不太适合.
```c
  /* 依次打开对应IO的中断 */  
  for (uint8_t i = 0; i < key_counts;i++)
  {
    err_code = gpio_set_intr_type((key_config+i)->key_number,GPIO_INTR_ANYEDGE);
    if(err_code != ESP_OK)
    {
      ESP_LOGI("user_key_init","gpio_set_intr_type is %d\n",err_code);
      return err_code;
    }
  } 
```
### 定时器
定时器在按键的应用是不可避免的,属于充分必要条件.那么要实现单击/长按/多击这里需要4个定时器,可能有人就会说 **"卧槽!搞个按键需要太多定时器了吧"**. 虽然需要4个定时器,但是它们基本上不会同时工作,你完全可以用一个定时器实现单击,长按,多击的计时.但是,因为ESP32自带freertos,所以有软件定时器可以直接用.因此,这个并不是什么问题,如果有老铁的定时器外设资源不够的话,你可以挪用一个定时器做为时基,采用链表的方法扩展出无限个软件定时器 **(这个后面有机会会出个专门的教程)**.

- 定时器1

  用于按键消抖,那么为什么要消抖呢?这是有历史原因的 **(使用触摸就不用消抖了,因为它也没办法抖:smirk:)**,具体原因如下所示:

```rst
按键消抖通常的按键所用开关为机械弹性开关，当机械触点断开、闭合时，由于机械触点的弹性作用，一个按键开关在闭合时不会马上稳定地接通，在断开时也不会一下子断开。因而在闭合及断开的瞬间均伴随有一连串的抖动，为了不产生这种现象而作的措施就是按键消抖.(出自<百度百科>)
```
![按键抖动过程](https://raw.githubusercontent.com/xiaolongba/picture/master/%E6%8C%89%E9%94%AE%E6%8A%96%E5%8A%A8.jpg)

```c
/* 填充消抖定时器所需要的相关函数 */
esp_timer_create_args_t esp_decounce_timer_args =
{
.callback  = after_key_decounce_cb,
.arg  =  NULL,
.dispatch_method  = ESP_TIMER_TASK,
.name  =  "esp_decounce_timer",
};
err_code =  esp_timer_create(&esp_decounce_timer_args,&gs_m_key_time_params.key_decounce_time_handle);
if (err_code != ESP_OK)
{
ESP_LOGI("user_key_init", "esp_decounce_timer is %d\n", err_code);
return err_code;
}
```

- 定时器2

  用于按键长按计时,当消抖完成之后检测到有按键按下则开启些定时器2用于长按计时.如果检测到按键释放了则停止定时器2计时.

  ![长按按键示意图](https://raw.githubusercontent.com/xiaolongba/picture/master/%E9%95%BF%E6%8C%89%E7%A4%BA%E6%84%8F%E5%9B%BE.svg?sanitize=true)

  从上图我们可以看出,当按下的时间大于等于5000ms,即认为是长按动作并执行长按的回调处理函数

- 定时器3&定时器4

  这两个定时器组合在一起,用于判断短按的整个动作是否完成,这里所谓的**短按整个动作**指的是按键按下并释放.那么怎么认为一个按键动作完成呢?并不是按下然后释放就算了,释放之后还需要再等待150ms左右,看看是否有新的按键按下,这样的目的主要是用于多击判断.
  ![单击/多击按键示意图](https://raw.githubusercontent.com/xiaolongba/picture/master/%E5%8D%95%E5%87%BB%20%E5%A4%9A%E5%87%BB.svg?sanitize=true)

  从上面的示意图,我们可以分析看出:

    - 一个完整的短按动作,从按下开始计时到释放这个过程不能大于250ms,否则认为这个短按动作无效    
    - 有效的短按动作完成之后,还需要再等待150ms用于检测有没有下一波的按键到来,如果没有则才是真正意义的短按动作完成,并且短按次数会自加,接下来就可以通过短按次数判断是多少击了
    - 如果是一个被识别为一个无效的短按动作,则将短按次数清0

```c
/** 
 * 消抖之后按键的具体处理函数
 * @param[in]   pin_no       :表示哪组按键
 * @param[in]   key_action   :表示按键的状态
 * @retval      NULL                            
 * @par         修改日志 
 *               Ver0.0.1:
                     Helon_Chan, 2018/06/16, 初始化版本\n 
 */
static void short_pressed_cb(uint8_t pin_no, uint8_t key_action)
{  
  int64_t current_time,difference_value;  
  static int64_t last_time = 0;
  struct timeval system_time;
  key_config_t *s_m_key_config = (gs_m_key_config+pin_no);  
  /* 获取此时的时间 */
  gettimeofday(&system_time,NULL);
  /* 转换成ms */
  current_time = system_time.tv_sec*1000+(system_time.tv_usec/1000);
  switch (key_action)
  {
  case APP_KEY_PUSH:
    esp_timer_stop(gs_m_key_time_params.short_press_time_handle);
    esp_timer_start_once(gs_m_key_time_params.long_press_time_handle,s_m_key_config->long_pressed_time);
    ESP_LOGI("short_pressed_cb","APP_KEY_PUSH\n");   
    break;
  case APP_KEY_RELEASE:
    ESP_LOGI("short_pressed_cb","APP_KEY_RELEASE\n");   
    esp_timer_stop(gs_m_key_time_params.long_press_time_handle);
    /* 如果按键的按下与释放的时间小于MULTI_PRESSED_TIMER则认为是整个短按按键动作完成 */
    ESP_LOGI("short_pressed_cb","current_time - last_time is %lld\n",current_time - last_time);
    // difference_value = current_time - last_time;   
    if ((current_time - last_time) < MULTI_PRESSED_TIMER)
    {
      s_m_key_config->short_pressed_counts++;
      ESP_LOGI("short_pressed_cb", "s_m_key_config->short_pressed_counts is %d\n", s_m_key_config->short_pressed_counts);
      // esp_timer_stop(gs_m_key_time_params.short_press_time_handle);
      esp_timer_start_once(gs_m_key_time_params.short_press_time_handle, SHORT_PRESS_DELAY_CHECK);
    }
    else
    {
      s_m_key_config->short_pressed_counts = 0;
    }
    break;
  }
  /* 处理完之后,将处理之前的值就是上一次的值了 */
  last_time = current_time;
}
```

### 按键相关配置参数
主要是给应用层的用户根据自己的按键硬件电路情况填充使用,即使有多个按键也可以直接填充,如下所示:
```c
/* 定义一个配置按键参数的结构体 */
typedef struct key_config
{
  uint8_t key_number;                                                               ///< 按键对应的GPIO口
  uint8_t active_state;                                                             ///< 指定按键按下是高电平有效,还是低电平有效
  uint8_t short_pressed_counts;                                                     ///< 保存按键短按的次数,用于多击的判断
  uint32_t long_pressed_time;                                                       ///< 按键长按时间,单位ms
  // gpio_pulldown_t pull_down_en;                                                     ///< 是否使能下拉
  // gpio_pullup_t pull_up_en;                                                         ///< 是否使能上拉
  // user_key_decounce_handler_t user_key_handler;                                     ///< 按键消抖后的处理函数
} key_config_t;

/* 填充需要配置的按键个数以及对应的相关参数 */
static key_config_t gs_m_key_config[BOARD_BUTTON_COUNT] =
{
  {BOARD_BUTTON,APP_KEY_ACTIVE_LOW,0,LONG_PRESSED_TIMER},
};
```

### 按键相关处理参数
这里主要用于填充消抖的时长以及长短按的回调处理函数,然后就可以直接在自己的应用层处理对应的长按或者短按的动作
```c
/** 
 * 用户的短按处理函数
 * @param[in]   key_num                 :短按按键对应GPIO口
 * @param[in]   short_pressed_counts    :短按按键对应GPIO口按下的次数,这里用不上
 * @retval      null
 * @par         修改日志 
 *               Ver0.0.1:
                     Helon_Chan, 2018/06/16, 初始化版本\n 
 */
void short_pressed_cb(uint8_t key_num,uint8_t *short_pressed_counts)
{  
  switch (key_num)
  {
    case BOARD_BUTTON:
      switch (*short_pressed_counts)
      {
      case 1:
        ESP_LOGI("short_pressed_cb","first press!!!\n");
        break;
      case 2:
        ESP_LOGI("short_pressed_cb","double press!!!\n");
        break;
      case 3:
        ESP_LOGI("short_pressed_cb","trible press!!!\n");
        break;
      case 4:
        ESP_LOGI("short_pressed_cb","quatary press!!!\n");
        break;
        // case ....:
        // break;
      }
      *short_pressed_counts = 0;
      break;
  
    default:
      break;
  }
}

/** 
 * 用户的长按处理函数
 * @param[in]   key_num                 :短按按键对应GPIO口
 * @param[in]   long_pressed_counts     :按键对应GPIO口按下的次数,这里用不上
 * @retval      null
 * @par         修改日志 
 *               Ver0.0.1:
                     Helon_Chan, 2018/06/16, 初始化版本\n 
 */
void long_pressed_cb(uint8_t key_num,uint8_t *long_pressed_counts)
{
  switch (key_num)
  {
    case BOARD_BUTTON:
      ESP_LOGI("long_pressed_cb","long press!!!\n");      
      break;
    default:
      break;
  }
}


/** 
 * 用户的按键初始化函数
 * @param[in]   null 
 * @retval      null
 * @par         修改日志 
 *               Ver0.0.1:
                     Helon_Chan, 2018/06/16, 初始化版本\n 
 */
void user_app_key_init(void)
{
    int32_t err_code;
    err_code = user_key_init(gs_m_key_config,BOARD_BUTTON_COUNT,DECOUNE_TIMER,long_pressed_cb,short_pressed_cb);
    ESP_LOGI("user_app_key_init","user_key_init is %d\n",err_code);
}
```

### 如何使用
小编作为最屌丝的一名工程师,深刻地明白以上的内容还远远不够,还必须要告诉看的人怎么使用才算是真正意义的教程.因为,据经验很多老铁不会马上一字一眼去看的,必须先看到效果才会认真阅读以上的内容,那么到底如何使用呢?
 - 在需要调用的地方<code>#include  "user_app.h"</code>
 - 在<code>main</code>函数中直接调用<code>user_key_init</code>并填充对应的形参即可
 
最终的具体源码已经放在[github](https://github.com/xiaolongba/wireless-tech/tree/master/%E8%BD%AF%E4%BB%B6/%E7%BA%A2%E6%97%AD%E6%97%A0%E7%BA%BF%E5%BC%80%E5%8F%91%E6%9D%BF%E5%AE%9E%E6%88%98%E6%95%99%E7%A8%8B%E5%AF%B9%E5%BA%94%E6%BA%90%E7%A0%81/ESP32/ESP32%E7%9A%84%E7%AC%AC%E4%BA%8C%E8%AF%BE%EF%BC%9A%E6%8C%89%E9%94%AE%E5%8D%95%E5%87%BB%E4%BB%A5%E5%8F%8A%E5%A4%9A%E5%87%BB%E7%9A%84%E5%AE%9E%E7%8E%B0/app)了
 
### 最终效果
![最终的效果](https://raw.githubusercontent.com/xiaolongba/picture/master/%E7%AC%AC%E4%BA%8C%E8%AF%BE%E6%9C%80%E7%BB%88%E6%95%88%E6%9E%9C.png)

## 技术交流
![QQ群](https://raw.githubusercontent.com/xiaolongba/picture/master/QQ%20Group.jpg)

**本文原创,转载请注明出处**

## 小贴士
由于微信不能插入外部链接,为了更好的阅读体验请点击左下角的**阅读原文**

