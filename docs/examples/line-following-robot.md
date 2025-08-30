# روبوت متتبع للخط باستخدام OpenCV | Line Following Robot using OpenCV

## 📖 وصف المشروع | Project Description

سنبني روبوت ذكي يستطيع تتبع خط أسود على أرضية بيضاء باستخدام تقنيات الرؤية الحاسوبية. هذا المشروع يجمع بين أساسيات الذكاء الاصطناعي والتحكم في الروبوتات.

We'll build an intelligent robot that can follow a black line on a white surface using computer vision techniques. This project combines AI fundamentals with robotics control.

## 🎯 المفاهيم المطبقة | Applied Concepts

- **الرؤية الحاسوبية**: معالجة الصور وكشف الحواف
- **التحكم التناسبي**: PID Controller للحركة السلسة
- **معالجة الصور الفورية**: Real-time image processing
- **التحكم في المحركات**: Motor control algorithms

## 🛠️ المتطلبات | Requirements

### الأجهزة | Hardware
```
- Raspberry Pi 4 أو حاسوب مع كاميرا USB
- كاميرا (دقة 480p على الأقل)
- روبوت مع محركات (أو محاكي)
- خط أسود على أرضية بيضاء للاختبار
```

### البرمجيات | Software
```bash
pip install opencv-python
pip install numpy
pip install matplotlib
pip install RPi.GPIO  # للـ Raspberry Pi فقط
```

## 💻 الكود الكامل | Complete Code

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
روبوت متتبع للخط باستخدام OpenCV
Line Following Robot using OpenCV

Author: AIBOMECH
License: MIT
"""

import cv2
import numpy as np
import time
import math

class LineFollowingRobot:
    def __init__(self, camera_index=0, debug_mode=True):
        """
        تهيئة روبوت تتبع الخط
        Initialize line following robot
        """
        # إعداد الكاميرا
        self.cap = cv2.VideoCapture(camera_index)
        self.cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
        self.cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)
        
        # معاملات التحكم PID
        self.kp = 0.5  # المعامل التناسبي
        self.ki = 0.0  # المعامل التكاملي
        self.kd = 0.3  # المعامل التفاضلي
        
        # متغيرات PID
        self.previous_error = 0
        self.integral = 0
        
        # سرعات المحركات
        self.base_speed = 100  # السرعة الأساسية
        self.max_speed = 255   # أقصى سرعة
        
        # وضع التصحيح
        self.debug_mode = debug_mode
        
        print("🤖 Line Following Robot Initialized")
        print("📹 Camera resolution: 640x480")
        print("🎮 PID Parameters: Kp={}, Ki={}, Kd={}".format(self.kp, self.ki, self.kd))

    def preprocess_image(self, frame):
        """
        معالجة الصورة لاستخراج الخط
        Preprocess image to extract line
        """
        # تحويل إلى رمادي
        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        
        # تطبيق blur لتقليل الضوضاء
        blurred = cv2.GaussianBlur(gray, (5, 5), 0)
        
        # تحويل ثنائي (Binary) لفصل الخط
        _, binary = cv2.threshold(blurred, 60, 255, cv2.THRESH_BINARY_INV)
        
        # تطبيق عمليات morfological للتنظيف
        kernel = np.ones((3, 3), np.uint8)
        binary = cv2.morphologyEx(binary, cv2.MORPH_CLOSE, kernel)
        binary = cv2.morphologyEx(binary, cv2.MORPH_OPEN, kernel)
        
        return binary

    def find_line_center(self, binary_image):
        """
        العثور على مركز الخط
        Find the center of the line
        """
        height, width = binary_image.shape
        
        # تحديد منطقة الاهتمام (النصف السفلي من الصورة)
        roi_height = int(height * 0.7)
        roi = binary_image[roi_height:height, :]
        
        # العثور على الكونتورات
        contours, _ = cv2.findContours(roi, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
        
        if not contours:
            return None, None
            
        # العثور على أكبر كونتور (الخط)
        largest_contour = max(contours, key=cv2.contourArea)
        
        # التحقق من حجم الكونتور
        if cv2.contourArea(largest_contour) < 100:
            return None, None
        
        # حساب مركز الكتلة
        M = cv2.moments(largest_contour)
        if M["m00"] != 0:
            cx = int(M["m10"] / M["m00"])
            cy = int(M["m01"] / M["m00"]) + roi_height
            return (cx, cy), largest_contour
        
        return None, None

    def calculate_pid(self, error):
        """
        حساب قيمة التحكم PID
        Calculate PID control value
        """
        # المعامل التناسبي
        proportional = error
        
        # المعامل التكاملي
        self.integral += error
        
        # المعامل التفاضلي
        derivative = error - self.previous_error
        
        # حساب الناتج النهائي
        output = (self.kp * proportional + 
                 self.ki * self.integral + 
                 self.kd * derivative)
        
        # حفظ الخطأ للمرة القادمة
        self.previous_error = error
        
        return output

    def control_motors(self, steering_value):
        """
        التحكم في المحركات بناءً على قيمة التوجيه
        Control motors based on steering value
        """
        # حساب سرعة كل محرك
        left_speed = self.base_speed - steering_value
        right_speed = self.base_speed + steering_value
        
        # تحديد السرعة ضمن النطاق المسموح
        left_speed = max(-self.max_speed, min(self.max_speed, left_speed))
        right_speed = max(-self.max_speed, min(self.max_speed, right_speed))
        
        # في التطبيق الحقيقي، ستكون هذه دوال التحكم في المحركات
        # set_motor_speed(left_motor, left_speed)
        # set_motor_speed(right_motor, right_speed)
        
        if self.debug_mode:
            print(f"🔧 Motor Speeds - Left: {left_speed:3.0f}, Right: {right_speed:3.0f}")
        
        return left_speed, right_speed

    def draw_debug_info(self, frame, center_point, error, steering_value):
        """
        رسم معلومات التصحيح على الإطار
        Draw debug information on frame
        """
        height, width = frame.shape[:2]
        center_x = width // 2
        
        # رسم خط المركز
        cv2.line(frame, (center_x, 0), (center_x, height), (0, 255, 0), 2)
        
        # رسم نقطة مركز الخط
        if center_point:
            cv2.circle(frame, center_point, 10, (0, 0, 255), -1)
            cv2.line(frame, (center_x, center_point[1]), center_point, (255, 0, 0), 3)
        
        # إضافة نص المعلومات
        cv2.putText(frame, f"Error: {error:.1f}", (10, 30), 
                   cv2.FONT_HERSHEY_SIMPLEX, 0.7, (255, 255, 255), 2)
        cv2.putText(frame, f"Steering: {steering_value:.1f}", (10, 60), 
                   cv2.FONT_HERSHEY_SIMPLEX, 0.7, (255, 255, 255), 2)
        cv2.putText(frame, f"PID: P={self.kp}, I={self.ki}, D={self.kd}", (10, 90), 
                   cv2.FONT_HERSHEY_SIMPLEX, 0.5, (255, 255, 255), 2)
        
        return frame

    def run(self):
        """
        الحلقة الرئيسية لتشغيل الروبوت
        Main loop to run the robot
        """
        print("🚀 Starting Line Following Robot...")
        print("Press 'q' to quit, 's' to save image, 'r' to reset PID")
        
        frame_count = 0
        start_time = time.time()
        
        try:
            while True:
                # قراءة الإطار
                ret, frame = self.cap.read()
                if not ret:
                    print("❌ Failed to read frame from camera")
                    break
                
                frame_count += 1
                
                # معالجة الصورة
                binary = self.preprocess_image(frame)
                
                # العثور على مركز الخط
                center_point, contour = self.find_line_center(binary)
                
                if center_point:
                    # حساب الخطأ (البعد عن المركز)
                    image_center = frame.shape[1] // 2
                    error = center_point[0] - image_center
                    
                    # حساب قيمة التحكم PID
                    steering_value = self.calculate_pid(error)
                    
                    # التحكم في المحركات
                    left_speed, right_speed = self.control_motors(steering_value)
                    
                else:
                    # لم يتم العثور على خط
                    error = 0
                    steering_value = 0
                    print("⚠️ Line not detected!")
                
                # رسم معلومات التصحيح
                if self.debug_mode:
                    debug_frame = self.draw_debug_info(frame.copy(), center_point, 
                                                     error, steering_value)
                    
                    # عرض الصور
                    cv2.imshow('Original', debug_frame)
                    cv2.imshow('Binary', binary)
                
                # التحكم في لوحة المفاتيح
                key = cv2.waitKey(1) & 0xFF
                if key == ord('q'):
                    break
                elif key == ord('s'):
                    # حفظ الصورة الحالية
                    timestamp = int(time.time())
                    cv2.imwrite(f'line_following_{timestamp}.jpg', frame)
                    print(f"💾 Image saved as line_following_{timestamp}.jpg")
                elif key == ord('r'):
                    # إعادة تعيين PID
                    self.previous_error = 0
                    self.integral = 0
                    print("🔄 PID values reset")
                
                # حساب FPS
                if frame_count % 30 == 0:
                    elapsed_time = time.time() - start_time
                    fps = frame_count / elapsed_time
                    print(f"📊 FPS: {fps:.1f}")
                    
        except KeyboardInterrupt:
            print("\n🛑 Stopped by user")
        
        finally:
            # تنظيف الموارد
            self.cap.release()
            cv2.destroyAllWindows()
            print("✅ Resources cleaned up")

    def tune_pid(self, kp=None, ki=None, kd=None):
        """
        ضبط معاملات PID
        Tune PID parameters
        """
        if kp is not None:
            self.kp = kp
        if ki is not None:
            self.ki = ki
        if kd is not None:
            self.kd = kd
        
        print(f"🎛️ PID Updated: Kp={self.kp}, Ki={self.ki}, Kd={self.kd}")

# مثال للاستخدام المتقدم
class AdvancedLineFollower(LineFollowingRobot):
    """
    نسخة متقدمة من متتبع الخط مع ميزات إضافية
    Advanced version with additional features
    """
    
    def __init__(self, camera_index=0, debug_mode=True):
        super().__init__(camera_index, debug_mode)
        
        # إضافة ذاكرة للحالات السابقة
        self.line_history = []
        self.max_history = 10
        
        # إضافة كشف المنعطفات
        self.turn_threshold = 100
        
    def detect_turn(self, error):
        """
        كشف المنعطفات الحادة
        Detect sharp turns
        """
        if abs(error) > self.turn_threshold:
            if error > 0:
                return "RIGHT_TURN"
            else:
                return "LEFT_TURN"
        return "STRAIGHT"
    
    def adaptive_speed(self, turn_type):
        """
        تعديل السرعة حسب نوع المنعطف
        Adaptive speed based on turn type
        """
        if turn_type == "STRAIGHT":
            self.base_speed = 100
        else:
            self.base_speed = 60  # تقليل السرعة في المنعطفات

if __name__ == "__main__":
    # إنشاء وتشغيل الروبوت
    robot = LineFollowingRobot(camera_index=0, debug_mode=True)
    
    # ضبط معاملات PID حسب الحاجة
    robot.tune_pid(kp=0.7, ki=0.0, kd=0.4)
    
    # تشغيل الروبوت
    robot.run()
```

## 🎛️ ضبط المعاملات | Parameter Tuning

### ضبط معاملات PID | PID Tuning

```python
# للحصول على استجابة سريعة
robot.tune_pid(kp=0.8, ki=0.0, kd=0.2)

# للحصول على حركة ناعمة
robot.tune_pid(kp=0.3, ki=0.1, kd=0.5)

# لتحسين الدقة
robot.tune_pid(kp=0.5, ki=0.05, kd=0.3)
```

### ضبط معالجة الصورة | Image Processing Tuning

```python
# لخطوط رفيعة
binary_threshold = 50

# لخطوط سميكة
binary_threshold = 80

# للبيئات المضيئة
binary_threshold = 100
```

## 📊 مراقبة الأداء | Performance Monitoring

```python
class PerformanceMonitor:
    def __init__(self):
        self.errors = []
        self.speeds = []
        self.timestamps = []
    
    def log_data(self, error, left_speed, right_speed):
        self.errors.append(error)
        self.speeds.append((left_speed, right_speed))
        self.timestamps.append(time.time())
    
    def plot_performance(self):
        import matplotlib.pyplot as plt
        
        plt.figure(figsize=(12, 8))
        
        # رسم الأخطاء
        plt.subplot(2, 1, 1)
        plt.plot(self.errors)
        plt.title('Line Following Error Over Time')
        plt.ylabel('Error (pixels)')
        
        # رسم سرعات المحركات
        plt.subplot(2, 1, 2)
        left_speeds = [s[0] for s in self.speeds]
        right_speeds = [s[1] for s in self.speeds]
        plt.plot(left_speeds, label='Left Motor')
        plt.plot(right_speeds, label='Right Motor')
        plt.title('Motor Speeds Over Time')
        plt.ylabel('Speed')
        plt.legend()
        
        plt.tight_layout()
        plt.show()
```

## 🚨 حل المشاكل | Troubleshooting

### مشاكل شائعة وحلولها | Common Issues and Solutions

```python
# مشكلة: الروبوت لا يتبع الخط
# الحل: ضبط threshold للتحويل الثنائي
def auto_tune_threshold(frame):
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    threshold_value = cv2.threshold(gray, 0, 255, 
                                   cv2.THRESH_BINARY + cv2.THRESH_OTSU)[0]
    return threshold_value

# مشكلة: حركة متذبذبة
# الحل: زيادة المعامل التفاضلي (Kd)
robot.tune_pid(kd=0.6)

# مشكلة: بطء الاستجابة
# الحل: زيادة المعامل التناسبي (Kp)
robot.tune_pid(kp=1.0)
```

## 🎯 تحديات للتطوير | Development Challenges

1. **إضافة كشف التقاطعات**: التعرف على التقاطعات واتخاذ القرارات
2. **تتبع خطوط ملونة**: تطوير النظام ليتبع خطوط بألوان مختلفة
3. **تجنب العوائق**: دمج مستشعرات المسافة لتجنب العوائق
4. **التعلم الآلي**: استخدام التعلم المعزز لتحسين الأداء

## 📚 موارد للتعلم | Learning Resources

- [OpenCV Python Tutorials](https://opencv-python-tutroals.readthedocs.io/)
- [PID Control Theory](https://en.wikipedia.org/wiki/PID_controller)
- [Computer Vision for Robotics](https://www.coursera.org/learn/robotics-perception)

---

**تمت بنجاح!** 🎉 لقد بنيت روبوت ذكي يتبع الخط باستخدام الذكاء الاصطناعي. هذا المشروع يعتبر أساساً قوياً لبناء أنظمة روبوتية أكثر تعقيداً.

**Success!** 🎉 You've built an intelligent line-following robot using AI. This project serves as a strong foundation for building more complex robotic systems.