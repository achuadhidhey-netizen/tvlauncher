name: Build TV launcher
on:
  push:
  workflow_dispatch:
permissions:
  contents: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: "8.9"
      - name: Create project files
        run: |
          mkdir -p app
          cat > app/build.gradle.kts <<'KT_EOF'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }
          android {
              namespace = "com.example.tvlauncher"
              compileSdk = 34
              defaultConfig {
                  applicationId = "com.example.tvlauncher"
                  minSdk = 21
                  targetSdk = 34
                  versionCode = 1
                  versionName = "1.0"
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
          }
          KT_EOF
          mkdir -p app/src/main
          cat > app/src/main/AndroidManifest.xml <<'KT_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-feature android:name="android.software.leanback" android:required="false" />
              <uses-feature android:name="android.hardware.touchscreen" android:required="false" />

              <queries>
                  <intent><action android:name="android.intent.action.MAIN" />
                      <category android:name="android.intent.category.LEANBACK_LAUNCHER" /></intent>
                  <intent><action android:name="android.intent.action.MAIN" />
                      <category android:name="android.intent.category.LAUNCHER" /></intent>
              </queries>

              <application
                  android:label="My Launcher"
                  android:theme="@android:style/Theme.DeviceDefault.NoActionBar">
                  <activity
                      android:name=".MainActivity"
                      android:exported="true"
                      android:launchMode="singleTask"
                      android:banner="@android:drawable/sym_def_app_icon">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.HOME" />
                          <category android:name="android.intent.category.DEFAULT" />
                      </intent-filter>
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>
              </application>
          </manifest>
          KT_EOF
          mkdir -p app/src/main/java/com/example/tvlauncher
          cat > app/src/main/java/com/example/tvlauncher/MainActivity.kt <<'KT_EOF'
          package com.example.tvlauncher

          import android.animation.ArgbEvaluator
          import android.animation.ValueAnimator
          import android.app.Activity
          import android.app.AlertDialog
          import android.content.Intent
          import android.content.SharedPreferences
          import android.content.pm.ResolveInfo
          import android.graphics.Bitmap
          import android.graphics.Canvas
          import android.graphics.Color
          import android.graphics.RenderEffect
          import android.graphics.Shader
          import android.graphics.drawable.Drawable
          import android.graphics.drawable.GradientDrawable
          import android.graphics.drawable.StateListDrawable
          import android.os.Build
          import android.os.Bundle
          import android.view.Gravity
          import android.view.View
          import android.widget.FrameLayout
          import android.widget.GridLayout
          import android.widget.ImageView
          import android.widget.LinearLayout
          import android.widget.ScrollView
          import android.widget.TextView

          class MainActivity : Activity() {

              private class WallTheme(val name: String, val c: IntArray)

              // c = [background top, background bottom, blob 1, blob 2, blob 3]
              private val themes = listOf(
                  WallTheme("Aurora", intArrayOf(0xFF0B1030.toInt(), 0xFF2A1050.toInt(), 0xCC3B82F6.toInt(), 0xCCEC4899.toInt(), 0xCC14B8A6.toInt())),
                  WallTheme("Sunset", intArrayOf(0xFF2B0F1E.toInt(), 0xFF4A1A10.toInt(), 0xCCF97316.toInt(), 0xCCEC4899.toInt(), 0xCCFACC15.toInt())),
                  WallTheme("Forest", intArrayOf(0xFF07201A.toInt(), 0xFF0F2E1F.toInt(), 0xCC10B981.toInt(), 0xCC84CC16.toInt(), 0xCC06B6D4.toInt())),
                  WallTheme("Midnight", intArrayOf(0xFF05060F.toInt(), 0xFF10131F.toInt(), 0xCC6366F1.toInt(), 0xCC312E81.toInt(), 0xCC0EA5E9.toInt())),
                  WallTheme("Rose", intArrayOf(0xFF2A0F2A.toInt(), 0xFF3A1030.toInt(), 0xCCF472B6.toInt(), 0xCCA855F7.toInt(), 0xCCFB7185.toInt()))
              )

              private lateinit var prefs: SharedPreferences
              private lateinit var allApps: List<ResolveInfo>
              private lateinit var grid: GridLayout
              private lateinit var bgDrawable: GradientDrawable
              private val blobDrawables = ArrayList<GradientDrawable>()
              private val iconColors = HashMap<String, IntArray>()
              private val evaluator = ArgbEvaluator()
              private var current = IntArray(5)
              private var animator: ValueAnimator? = null

              private fun dp(v: Int) = (v * resources.displayMetrics.density).toInt()

              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  prefs = getSharedPreferences("launcher", MODE_PRIVATE)
                  val pm = packageManager

                  fun query(category: String) = pm.queryIntentActivities(
                      Intent(Intent.ACTION_MAIN).addCategory(category), 0
                  )
                  allApps = (query(Intent.CATEGORY_LEANBACK_LAUNCHER) + query(Intent.CATEGORY_LAUNCHER))
                      .distinctBy { it.activityInfo.packageName }
                      .filter { it.activityInfo.packageName != packageName }
                      .sortedBy { it.loadLabel(pm).toString().lowercase() }

                  // ---------- Wallpaper layers ----------
                  bgDrawable = GradientDrawable(GradientDrawable.Orientation.TL_BR, intArrayOf(0, 0))
                  val root = FrameLayout(this).apply { background = bgDrawable }
                  val blobLayer = FrameLayout(this)
                  val sizes = intArrayOf(dp(520), dp(560), dp(480))
                  val xs = intArrayOf(dp(-120), dp(700), dp(250))
                  val ys = intArrayOf(dp(-100), dp(80), dp(380))
                  for (i in 0..2) {
                      val gd = GradientDrawable().apply {
                          gradientType = GradientDrawable.RADIAL_GRADIENT
                          gradientRadius = sizes[i] / 2f
                      }
                      blobDrawables.add(gd)
                      blobLayer.addView(View(this).apply {
                          background = gd
                          layoutParams = FrameLayout.LayoutParams(sizes[i], sizes[i]).apply {
                              leftMargin = xs[i]; topMargin = ys[i]
                          }
                      })
                  }
                  if (Build.VERSION.SDK_INT >= 31) {
                      blobLayer.setRenderEffect(RenderEffect.createBlurEffect(120f, 120f, Shader.TileMode.CLAMP))
                  }
                  root.addView(blobLayer, FrameLayout.LayoutParams(-1, -1))
                  current = globalColors().copyOf()
                  paint()

                  // ---------- Header with two buttons ----------
                  val header = LinearLayout(this).apply {
                      orientation = LinearLayout.HORIZONTAL
                      gravity = Gravity.CENTER_VERTICAL
                      setPadding(dp(48), dp(32), dp(48), 0)
                  }
                  header.addView(TextView(this).apply {
                      text = "Home"
                      setTextColor(Color.WHITE)
                      textSize = 28f
                  }, LinearLayout.LayoutParams(0, -2, 1f))
                  header.addView(glassButton("Wallpaper") { showWallpaperDialog() })
                  header.addView(glassButton("Manage apps") { showManageDialog() })

                  grid = GridLayout(this).apply {
                      columnCount = 5
                      setPadding(dp(40), dp(24), dp(40), dp(40))
                      clipChildren = false
                      clipToPadding = false
                  }
                  val scroll = ScrollView(this).apply {
                      isFillViewport = true
                      clipChildren = false
                      clipToPadding = false
                      addView(grid)
                  }
                  val content = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL }
                  content.addView(header, LinearLayout.LayoutParams(-1, -2))
                  content.addView(scroll, LinearLayout.LayoutParams(-1, 0, 1f))
                  root.addView(content, FrameLayout.LayoutParams(-1, -1))
                  setContentView(root)

                  refreshGrid()
              }

              // ---------- App grid ----------
              private fun refreshGrid() {
                  val pm = packageManager
                  val hidden = prefs.getStringSet("hidden", emptySet())!!
                  val pinned = prefs.getStringSet("pinned", emptySet())!!
                  val apps = allApps
                      .filter { it.activityInfo.packageName !in hidden }
                      .sortedByDescending { it.activityInfo.packageName in pinned }

                  grid.removeAllViews()
                  for (info in apps) {
                      val pkg = info.activityInfo.packageName
                      val icon = info.loadIcon(pm)
                      val isPinned = pkg in pinned
                      val item = LinearLayout(this).apply {
                          orientation = LinearLayout.VERTICAL
                          gravity = Gravity.CENTER
                          setPadding(dp(14), dp(18), dp(14), dp(14))
                          isFocusable = true
                          isClickable = true
                          background = glassBackground()
                          layoutParams = GridLayout.LayoutParams().apply {
                              width = 0
                              height = GridLayout.LayoutParams.WRAP_CONTENT
                              columnSpec = GridLayout.spec(GridLayout.UNDEFINED, 1f)
                              setMargins(dp(8), dp(8), dp(8), dp(8))
                          }
                          setOnFocusChangeListener { v, focused ->
                              val s = if (focused) 1.08f else 1f
                              v.animat
