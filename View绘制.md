

### Paint

设置颜色

* setColor/ARGB 

  * ```java
    paint.setColor(Color.parseColor("#009688"));
    paint.setARGB(100, 255, 0, 0);
    ```

* setShader(Shader shader) 

  * LinearGradient
  * RadialGradient 
  * SweepGradient 
  * BitmapShader
  * ComposeShader(Shader shaderA, Shader shaderB, **PorterDuff.Mode** mode)

* setColorFilter()

  * LightingColorFilter  是用来模拟简单的光照效果的
  * PorterDuffColorFilter 
  * ColorMatrixColorFilter

