# البدء بـ ROS والذكاء الاصطناعي | Getting Started with ROS and AI

## 📖 نظرة عامة | Overview

في هذا الدرس، ستتعلم كيفية دمج تقنيات الذكاء الاصطناعي مع نظام تشغيل الروبوتات (ROS). سنبني روبوت بسيط يستخدم الرؤية الحاسوبية لتتبع الكائنات.

In this tutorial, you'll learn how to integrate AI technologies with Robot Operating System (ROS). We'll build a simple robot that uses computer vision to track objects.

## 🎯 أهداف التعلم | Learning Objectives

بنهاية هذا الدرس ستتمكن من:
- فهم أساسيات ROS وهيكليته
- إنشاء عقد ROS للرؤية الحاسوبية  
- دمج OpenCV مع ROS
- تطبيق خوارزميات تتبع الكائنات
- التحكم في حركة الروبوت بناءً على الرؤية

## 🛠️ المتطلبات | Prerequisites

### البرمجيات المطلوبة | Required Software
```bash
# تثبيت ROS Noetic (Ubuntu 20.04)
sudo apt update
sudo apt install ros-noetic-desktop-full

# تثبيت مكتبات Python
pip install opencv-python
pip install numpy
pip install rospy
pip install sensor_msgs
pip install geometry_msgs
```

### المعرفة المطلوبة | Required Knowledge
- أساسيات Python
- مفاهيم أساسية في الرؤية الحاسوبية
- فهم بسيط لأنظمة التحكم

## 📋 خطوات التطبيق | Implementation Steps

### الخطوة 1: إعداد بيئة ROS | Step 1: Setting Up ROS Environment

```bash
# إعداد متغيرات البيئة
echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc
source ~/.bashrc

# إنشاء workspace جديد
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws/
catkin_make

# إعداد workspace
echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### الخطوة 2: إنشاء Package | Step 2: Creating ROS Package

```bash
# الانتقال إلى مجلد src
cd ~/catkin_ws/src

# إنشاء package جديد
catkin_create_pkg ai_robot_tutorial rospy std_msgs sensor_msgs geometry_msgs cv_bridge

# بناء Package
cd ~/catkin_ws
catkin_make
```

### الخطوة 3: كتابة عقدة الرؤية الحاسوبية | Step 3: Computer Vision Node

إنشاء ملف `object_tracker.py`:

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import rospy
import cv2
import numpy as np
from sensor_msgs.msg import Image
from geometry_msgs.msg import Twist
from cv_bridge import CvBridge, CvBridgeError

class ObjectTracker:
    def __init__(self):
        # تهيئة العقدة
        rospy.init_node('object_tracker', anonymous=True)
        
        # إعداد المتغيرات
        self.bridge = CvBridge()
        self.image_sub = rospy.Subscriber("/camera/image_raw", Image, self.image_callback)
        self.cmd_vel_pub = rospy.Publisher('/cmd_vel', Twist, queue_size=10)
        
        # معاملات تتبع اللون (أحمر)
        self.lower_red = np.array([0, 50, 50])
        self.upper_red = np.array([10, 255, 255])
        
        # معاملات التحكم
        self.linear_speed = 0.3
        self.angular_speed = 0.5
        
        rospy.loginfo("Object Tracker Node Started")

    def image_callback(self, data):
        try:
            # تحويل صورة ROS إلى OpenCV
            cv_image = self.bridge.imgmsg_to_cv2(data, "bgr8")
        except CvBridgeError as e:
            rospy.logerr(e)
            return

        # معالجة الصورة
        self.process_image(cv_image)

    def process_image(self, image):
        # تحويل إلى HSV
        hsv = cv2.cvtColor(image, cv2.COLOR_BGR2HSV)
        
        # إنشاء قناع للكائن الأحمر
        mask = cv2.inRange(hsv, self.lower_red, self.upper_red)
        
        # تطبيق عمليات التنظيف
        mask = cv2.erode(mask, None, iterations=2)
        mask = cv2.dilate(mask, None, iterations=2)
        
        # العثور على الكونتورات
        contours, _ = cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
        
        if contours:
            # العثور على أكبر كونتور
            largest_contour = max(contours, key=cv2.contourArea)
            
            # حساب المركز
            M = cv2.moments(largest_contour)
            if M["m00"] != 0:
                cx = int(M["m10"] / M["m00"])
                cy = int(M["m01"] / M["m00"])
                
                # التحكم في الروبوت
                self.control_robot(cx, image.shape[1])
                
                # رسم دائرة على المركز
                cv2.circle(image, (cx, cy), 10, (0, 255, 0), -1)
        
        else:
            # إيقاف الروبوت إذا لم يوجد كائن
            self.stop_robot()
        
        # عرض الصورة
        cv2.imshow("Object Tracker", image)
        cv2.imshow("Mask", mask)
        cv2.waitKey(1)

    def control_robot(self, object_x, image_width):
        """التحكم في حركة الروبوت بناءً على موقع الكائن"""
        twist = Twist()
        
        # حساب الخطأ (البعد عن المركز)
        center_x = image_width // 2
        error = object_x - center_x
        
        # تحديد السرعة الزاوية بناءً على الخطأ
        if abs(error) > 30:  # تحمل الخطأ
            twist.angular.z = -error * 0.01  # تناسب الاستجابة مع الخطأ
            twist.linear.x = 0.1  # سرعة بطيئة أثناء التوجه
        else:
            # الكائن في المركز، التحرك للأمام
            twist.linear.x = self.linear_speed
            twist.angular.z = 0.0
        
        # نشر الأمر
        self.cmd_vel_pub.publish(twist)

    def stop_robot(self):
        """إيقاف الروبوت"""
        twist = Twist()
        twist.linear.x = 0.0
        twist.angular.z = 0.0
        self.cmd_vel_pub.publish(twist)

    def shutdown(self):
        """إغلاق العقدة بأمان"""
        rospy.loginfo("Shutting down Object Tracker")
        self.stop_robot()
        cv2.destroyAllWindows()

if __name__ == '__main__':
    try:
        tracker = ObjectTracker()
        rospy.on_shutdown(tracker.shutdown)
        rospy.spin()
    except rospy.ROSInterruptException:
        pass
```

### الخطوة 4: إنشاء Launch File | Step 4: Creating Launch File

إنشاء ملف `object_tracking.launch`:

```xml
<launch>
    <!-- تشغيل عقدة الكاميرا -->
    <node name="usb_cam" pkg="usb_cam" type="usb_cam_node" output="screen">
        <param name="video_device" value="/dev/video0" />
        <param name="image_width" value="640" />
        <param name="image_height" value="480" />
        <param name="pixel_format" value="yuyv" />
        <param name="camera_frame_id" value="usb_cam" />
        <param name="io_method" value="mmap"/>
    </node>

    <!-- تشغيل عقدة تتبع الكائنات -->
    <node name="object_tracker" pkg="ai_robot_tutorial" type="object_tracker.py" output="screen"/>

    <!-- تشغيل محاكي الروبوت (اختياري) -->
    <include file="$(find turtlebot3_gazebo)/launch/turtlebot3_world.launch" if="$(eval arg('simulation') == true)"/>
</launch>
```

### الخطوة 5: تشغيل النظام | Step 5: Running the System

```bash
# جعل الملف قابل للتنفيذ
chmod +x ~/catkin_ws/src/ai_robot_tutorial/scripts/object_tracker.py

# بناء Package
cd ~/catkin_ws
catkin_make

# تشغيل roscore
roscore &

# تشغيل النظام
roslaunch ai_robot_tutorial object_tracking.launch
```

## 🔧 تحسينات متقدمة | Advanced Improvements

### تحسين دقة التتبع | Improving Tracking Accuracy

```python
# إضافة فلتر Kalman للتنبؤ بحركة الكائن
class KalmanTracker:
    def __init__(self):
        self.kalman = cv2.KalmanFilter(4, 2)
        self.kalman.measurementMatrix = np.array([[1, 0, 0, 0],
                                                  [0, 1, 0, 0]], np.float32)
        self.kalman.transitionMatrix = np.array([[1, 0, 1, 0],
                                                 [0, 1, 0, 1],
                                                 [0, 0, 1, 0],
                                                 [0, 0, 0, 1]], np.float32)
        self.kalman.processNoiseCov = 0.03 * np.eye(4, dtype=np.float32)

    def predict(self):
        return self.kalman.predict()

    def update(self, measurement):
        self.kalman.correct(measurement)
```

### إضافة تعدد الكائنات | Multi-Object Tracking

```python
def track_multiple_objects(self, contours):
    """تتبع عدة كائنات في نفس الوقت"""
    objects = []
    for contour in contours:
        if cv2.contourArea(contour) > 500:  # تصفية الكائنات الصغيرة
            M = cv2.moments(contour)
            if M["m00"] != 0:
                cx = int(M["m10"] / M["m00"])
                cy = int(M["m01"] / M["m00"])
                objects.append((cx, cy))
    
    return objects
```

## 📊 اختبار وتقييم الأداء | Testing and Performance Evaluation

### معايير التقييم | Evaluation Metrics

```python
class PerformanceMonitor:
    def __init__(self):
        self.tracking_accuracy = 0.0
        self.response_time = 0.0
        self.lost_frames = 0
        
    def calculate_accuracy(self, predicted_pos, actual_pos):
        """حساب دقة التتبع"""
        distance = np.sqrt((predicted_pos[0] - actual_pos[0])**2 + 
                          (predicted_pos[1] - actual_pos[1])**2)
        return 1.0 / (1.0 + distance)
    
    def measure_response_time(self, start_time):
        """قياس زمن الاستجابة"""
        return rospy.Time.now().to_sec() - start_time
```

## 🚨 حل المشاكل الشائعة | Troubleshooting

### مشكلة عدم اكتشاف الكاميرا | Camera Detection Issues
```bash
# التحقق من الكاميرات المتاحة
ls /dev/video*

# اختبار الكاميرا
cheese  # أو أي تطبيق كاميرا
```

### مشكلة بطء المعالجة | Slow Processing Issues
```python
# تقليل دقة الصورة
resized_image = cv2.resize(image, (320, 240))

# معالجة كل إطار ثاني
if self.frame_count % 2 == 0:
    self.process_image(image)
self.frame_count += 1
```

## 📚 موارد إضافية | Additional Resources

### مراجع مفيدة | Useful References
- [ROS Tutorials](http://wiki.ros.org/ROS/Tutorials)
- [OpenCV Python Tutorials](https://opencv-python-tutroals.readthedocs.io/)
- [Computer Vision for Robotics](https://www.robotics.utah.edu/vision/)

### مشاريع مشابهة | Similar Projects
- [TurtleBot Object Following](https://github.com/markwsilliman/turtlebot/)
- [ROS Navigation Stack](http://wiki.ros.org/navigation)

## 🎯 التحديات العملية | Practical Challenges

1. **تحسين تتبع الكائنات في ظروف إضاءة مختلفة**
2. **إضافة تجنب العوائق**
3. **تطبيق تعلم معزز للتحكم الذكي**
4. **دمج مستشعرات إضافية (ليدار، IMU)**

---

**تهانينا!** 🎉 لقد أكملت بنجاح أول مشروع ROS مع الذكاء الاصطناعي. هذا الأساس سيمكنك من بناء أنظمة روبوتية أكثر تعقيداً وذكاءً.

**Congratulations!** 🎉 You've successfully completed your first ROS AI project. This foundation will enable you to build more complex and intelligent robotic systems.