## #2.验证码

### #2.1源码

#### #2.1.1成员变量

```java
private int width = 100;// 生成验证码图片的宽度
private int height = 30;// 生成验证码图片的高度
private String[] fontNames = { "宋体", "楷体", "隶书", "微软雅黑" };
private Color bgColor = new Color(255, 255, 255);// 定义验证码图片的背景颜色为白色
private Random random = new Random();
private String codes = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
private String text;// 记录随机字符串
```

这里规定了图片的格式、验证码的字体集合（fontNames）、图片背景颜色，一个随机数生成器（用于从codes中随机获取字符）、验证码的取数集合（codes）还有此次选出的验证码字符串

#### #2.1.2构造函数

```java
private Color randomColor() {
   int red = random.nextInt(150);
   int green = random.nextInt(150);
   int blue = random.nextInt(150);
   return new Color(red, green, blue);
}
```

用于设定Color对象的rgb参数

#### #2.1.3字体

```java
private Font randomFont() {
   String name = fontNames[random.nextInt(fontNames.length)];
   int style = random.nextInt(4);
   int size = random.nextInt(5) + 24;
   return new Font(name, style, size);
}
```

用于从fontNames中随机获取一个字体

#### #2.1.4随机获取一个字符

```java
private char randomChar() {
   return codes.charAt(random.nextInt(codes.length()));
}
```

用于从验证码取值集合codes中随机获取一个字符

#### #2.1.5创建图的缓冲区对象

```java
private BufferedImage createImage() {
   BufferedImage image = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
   Graphics2D g2 = (Graphics2D) image.getGraphics();
   g2.setColor(bgColor);// 设置验证码图片的背景颜色
   g2.fillRect(0, 0, width, height);
   return image;
}
```

根据默认参数（背景颜色、尺寸）初始化一个BufferedImage对象并返回

#### #2.1.6获取图的缓冲区对象

```java
public BufferedImage getImage() {
   BufferedImage image = createImage();
   Graphics2D g2 = (Graphics2D) image.getGraphics();
   StringBuffer sb = new StringBuffer();
   for (int i = 0; i < 4; i++) {
      String s = randomChar() + "";
      sb.append(s);
      g2.setColor(randomColor());
      g2.setFont(randomFont());
      float x = i * width * 1.0f / 4;
      g2.drawString(s, x, height - 8);
   }
   this.text = sb.toString();
   drawLine(image);
   return image;
}
```

首先调用createImage得到一个初始化好的图，之后进入for循环得到四个字符，字符串备份存入sb，每次获取到的字符会按4等分位置从左至右顺序绘制到image对象中。最终调用drewLine方法

#### #2.1.7绘制干扰线

```java
private void drawLine(BufferedImage image) {
   Graphics2D g2 = (Graphics2D) image.getGraphics();
   int num = 5;
   for (int i = 0; i < num; i++) {
      int x1 = random.nextInt(width);
      int y1 = random.nextInt(height);
      int x2 = random.nextInt(width);
      int y2 = random.nextInt(height);
      g2.setColor(randomColor());
      g2.setStroke(new BasicStroke(1.5f));
      g2.drawLine(x1, y1, x2, y2);
   }
}
```

以图的范围为界限，随机获取到四个点坐标，两两一组构成一条线，设定好线条的颜色和宽度后，将干扰线绘制在image对象上

#### #2.1.8输出到前端

```java
public static void output(BufferedImage image, OutputStream out) throws IOException {
   ImageIO.write(image, "JPEG", out);
}
```

通过给定的out对象（由resp.getOutputStream()得到）和刚绘制好的image对象，调用ImageIO的静态方法将字节流写到页面上

### #2.2调用点

```java
@RestController
public class LoginController {
    @GetMapping("/verifyCode")
    public void verifyCode(HttpServletRequest request, HttpServletResponse resp) throws IOException {
        VerificationCode code = new VerificationCode();
        BufferedImage image = code.getImage();
        String text = code.getText();
        HttpSession session = request.getSession(true);
        session.setAttribute("verify_code", text);
        VerificationCode.output(image,resp.getOutputStream());
    }
}
```

当发出/verifyCode请求时，会调用此方法将新生成的验证码输出到前端