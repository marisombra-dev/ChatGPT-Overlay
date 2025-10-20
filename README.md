# ChatGPT Avatar Overlay

An Android overlay app that displays an animated avatar during ChatGPT voice conversations.

## Features
- 🎭 Automatically appears when ChatGPT voice mode is active
- 👆 Tap to start/stop lip-sync animation
- 🎯 Drag to reposition anywhere on screen
- 🔄 Double-tap to cycle through display modes (corner, sidebar, ambient)
- 👻 Transparent background - avatar floats over ChatGPT

## How It Works
The app monitors your device's audio output. When it detects that voice mode is active (audio playing), the avatar overlay appears. You control the animation manually by tapping - this way it only animates when the LLM is actually speaking to you, not when you're speaking.

## Installation

### Prerequisites
- Android 8.0 (API 26) or higher
- Android Studio (for building from source)

### Build & Install
1. Clone this repository
2. Open in Android Studio
3. Build the APK:
cd D:\chatgpt-overlay

# 1. Remove debug background
app\src\main\java\com\overlay\chatgpt\OverlayService.kt = "app\src\main\java\com\overlay\chatgpt\OverlayService.kt"
package com.overlay.chatgpt

import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.app.Service
import android.content.Context
import android.content.Intent
import android.content.SharedPreferences
import android.graphics.PixelFormat
import android.media.AudioManager
import android.media.AudioPlaybackConfiguration
import android.os.Build
import android.os.IBinder
import android.util.Log
import android.view.*
import android.view.animation.AccelerateDecelerateInterpolator
import android.widget.FrameLayout
import android.widget.ImageView
import androidx.core.app.NotificationCompat
import com.bumptech.glide.Glide
import com.bumptech.glide.load.DataSource
import com.bumptech.glide.load.engine.GlideException
import com.bumptech.glide.load.resource.gif.GifDrawable
import com.bumptech.glide.request.RequestListener
import com.bumptech.glide.request.target.Target
import kotlinx.coroutines.*
import kotlin.math.abs

class OverlayService : Service() {
    
    private lateinit var windowManager: WindowManager
    private var overlayView: FrameLayout? = null
    private var avatarView: ImageView? = null
    private lateinit var prefs: SharedPreferences
    private lateinit var audioManager: AudioManager
    
    private val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())
    private var idleAnimationJob: Job? = null
    private var monitorJob: Job? = null
    private var audioMonitorJob: Job? = null
    
    private var gifDrawable: GifDrawable? = null
    
    private val audioPlaybackCallback = object : AudioManager.AudioPlaybackCallback() {
        override fun onPlaybackConfigChanged(configs: MutableList<AudioPlaybackConfiguration>?) {
            super.onPlaybackConfigChanged(configs)
            
            val isPlaying = configs?.isNotEmpty() == true
            
            scope.launch {
                if (isPlaying && !isVoiceModeActive) {
                    isVoiceModeActive = true
                    showAvatar()
                } else if (!isPlaying && isVoiceModeActive) {
                    isVoiceModeActive = false
                    hideAvatar()
                }
            }
        }
    }
    
    enum class PositionMode {
        TOP_CORNER, SIDEBAR, AMBIENT, CUSTOM
    }
    
    private var currentMode = PositionMode.TOP_CORNER
    private var isInteracting = false
    private var isVoiceModeActive = false
    private var customX = 0
    private var customY = 0
    private var isViewAttached = false
    
    companion object {
        private const val TAG = "OverlayService"
        private const val PREFS_NAME = "overlay_prefs"
        private const val KEY_LAST_MODE = "last_mode"
        private const val KEY_CUSTOM_X = "custom_x"
        private const val KEY_CUSTOM_Y = "custom_y"
        private const val KEY_CUSTOM_WIDTH = "custom_width"
        private const val KEY_CUSTOM_HEIGHT = "custom_height"
        private const val CHANNEL_ID = "overlay_service_channel"
        private const val NOTIFICATION_ID = 1
    }
    
    override fun onCreate() {
        super.onCreate()
        
        startForegroundService()
        
        windowManager = getSystemService(Context.WINDOW_SERVICE) as WindowManager
        audioManager = getSystemService(Context.AUDIO_SERVICE) as AudioManager
        prefs = getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE)
        
        restoreLastState()
        createOverlay()
        startIdleAnimation()
        startMonitoring()
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            audioManager.registerAudioPlaybackCallback(audioPlaybackCallback, null)
        }
        
        startAudioMonitoring()
    }
    
    private fun startForegroundService() {
        createNotificationChannel()
        
        val notificationIntent = Intent(this, OverlayControlActivity::class.java)
        val pendingIntent = PendingIntent.getActivity(
            this, 0, notificationIntent,
            PendingIntent.FLAG_IMMUTABLE
        )
        
        val notification = NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("Avatar Overlay")
            .setContentText("Tap to start, tap again to stop")
            .setSmallIcon(android.R.drawable.ic_dialog_info)
            .setContentIntent(pendingIntent)
            .build()
        
        startForeground(NOTIFICATION_ID, notification)
    }
    
    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val serviceChannel = NotificationChannel(
                CHANNEL_ID,
                "Overlay Service",
                NotificationManager.IMPORTANCE_LOW
            )
            val manager = getSystemService(NotificationManager::class.java)
            manager.createNotificationChannel(serviceChannel)
        }
    }
    
    private fun startAudioMonitoring() {
        audioMonitorJob?.cancel()
        audioMonitorJob = scope.launch {
            while (isActive) {
                val isMusicActive = audioManager.isMusicActive
                val hasActivePlayback = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                    audioManager.activePlaybackConfigurations.isNotEmpty()
                } else {
                    false
                }
                
                val isAudioPlaying = isMusicActive || hasActivePlayback
                
                if (isAudioPlaying && !isVoiceModeActive) {
                    isVoiceModeActive = true
                    showAvatar()
                } else if (!isAudioPlaying && isVoiceModeActive) {
                    isVoiceModeActive = false
                    hideAvatar()
                }
                
                delay(300)
            }
        }
    }
    
    private fun showAvatar() {
        avatarView?.animate()
            ?.alpha(0.85f)
            ?.setDuration(200)
            ?.start()
    }
    
    private fun hideAvatar() {
        gifDrawable = null
        avatarView?.setImageDrawable(null)
        
        avatarView?.animate()
            ?.alpha(0.0f)
            ?.setDuration(200)
            ?.start()
    }
    
    private fun toggleAnimation() {
        if (gifDrawable == null) {
            // Load and play GIF
            loadAndPlayGif()
        } else {
            // Stop by completely removing the GIF
            gifDrawable = null
            avatarView?.setImageDrawable(null)
        }
    }
    
    private fun loadAndPlayGif() {
        Glide.with(this)
            .asGif()
            .load(R.raw.avatar_lipsync)
            .listener(object : RequestListener<GifDrawable> {
                override fun onLoadFailed(
                    e: GlideException?,
                    model: Any?,
                    target: Target<GifDrawable>,
                    isFirstResource: Boolean
                ): Boolean {
                    return false
                }
                
                override fun onResourceReady(
                    resource: GifDrawable,
                    model: Any,
                    target: Target<GifDrawable>?,
                    dataSource: DataSource,
                    isFirstResource: Boolean
                ): Boolean {
                    gifDrawable = resource
                    resource.setLoopCount(GifDrawable.LOOP_FOREVER)
                    return false
                }
            })
            .into(avatarView!!)
    }
    
    private fun startMonitoring() {
        monitorJob?.cancel()
        monitorJob = scope.launch {
            while (isActive) {
                delay(500)
                
                if (isViewAttached && overlayView?.windowToken == null) {
                    isViewAttached = false
                    
                    delay(100)
                    try {
                        createOverlay()
                        if (isVoiceModeActive) {
                            showAvatar()
                        }
                    } catch (e: Exception) {
                        Log.e(TAG, "Failed to re-attach: ${e.message}")
                    }
                }
            }
        }
    }
    
    private fun restoreLastState() {
        val savedMode = prefs.getString(KEY_LAST_MODE, PositionMode.TOP_CORNER.name)
        currentMode = try {
            PositionMode.valueOf(savedMode ?: PositionMode.TOP_CORNER.name)
        } catch (e: Exception) {
            PositionMode.TOP_CORNER
        }
        
        if (currentMode == PositionMode.CUSTOM) {
            customX = prefs.getInt(KEY_CUSTOM_X, 50)
            customY = prefs.getInt(KEY_CUSTOM_Y, 100)
        }
    }
    
    private fun saveCurrentState() {
        prefs.edit().apply {
            putString(KEY_LAST_MODE, currentMode.name)
            if (currentMode == PositionMode.CUSTOM) {
                putInt(KEY_CUSTOM_X, customX)
                putInt(KEY_CUSTOM_Y, customY)
                val params = overlayView?.layoutParams as? WindowManager.LayoutParams
                params?.let {
                    putInt(KEY_CUSTOM_WIDTH, avatarView?.width ?: 320)
                    putInt(KEY_CUSTOM_HEIGHT, avatarView?.height ?: 320)
                }
            }
            apply()
        }
    }
    
    private fun createOverlay() {
        try {
            if (overlayView != null && isViewAttached) {
                windowManager.removeView(overlayView)
                isViewAttached = false
            }
        } catch (e: Exception) {
            Log.e(TAG, "Error removing old view: ${e.message}")
        }
        
        overlayView = FrameLayout(this).apply {
            layoutParams = FrameLayout.LayoutParams(
                FrameLayout.LayoutParams.WRAP_CONTENT,
                FrameLayout.LayoutParams.WRAP_CONTENT
            )
        }
        
        val savedWidth = prefs.getInt(KEY_CUSTOM_WIDTH, 320)
        val savedHeight = prefs.getInt(KEY_CUSTOM_HEIGHT, 320)
        
        avatarView = ImageView(this).apply {
            layoutParams = FrameLayout.LayoutParams(savedWidth, savedHeight)
            scaleType = ImageView.ScaleType.FIT_CENTER
            alpha = 0.0f
        }
        
        overlayView?.addView(avatarView)
        
        val params = createWindowParams()
        setupTouchHandling(params)
        
        try {
            windowManager.addView(overlayView, params)
            isViewAttached = true
            
            applyPositionMode(params, animate = false)
        } catch (e: Exception) {
            Log.e(TAG, "Error creating overlay: ${e.message}")
            e.printStackTrace()
            isViewAttached = false
        }
    }
    
    private fun createWindowParams(): WindowManager.LayoutParams {
        val layoutType = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
        } else {
            @Suppress("DEPRECATION")
            WindowManager.LayoutParams.TYPE_PHONE
        }
        
        return WindowManager.LayoutParams(
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.WRAP_CONTENT,
            layoutType,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
                    WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL or
                    WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH or
                    WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS,
            PixelFormat.TRANSLUCENT
        ).apply {
            gravity = Gravity.TOP or Gravity.START
            x = if (currentMode == PositionMode.CUSTOM) customX else 50
            y = if (currentMode == PositionMode.CUSTOM) customY else 100
        }
    }
    
    private fun startIdleAnimation() {
        idleAnimationJob?.cancel()
        idleAnimationJob = scope.launch {
            while (isActive) {
                delay(3000)
                
                if (!isInteracting && isViewAttached && gifDrawable == null && isVoiceModeActive) {
                    avatarView?.animate()
                        ?.alpha(0.7f)
                        ?.setDuration(1500)
                        ?.setInterpolator(AccelerateDecelerateInterpolator())
                        ?.withEndAction {
                            avatarView?.animate()
                                ?.alpha(0.85f)
                                ?.setDuration(1500)
                                ?.setInterpolator(AccelerateDecelerateInterpolator())
                                ?.start()
                        }
                        ?.start()
                }
            }
        }
    }
    
    private fun setupTouchHandling(params: WindowManager.LayoutParams) {
        var initialX = 0
        var initialY = 0
        var initialTouchX = 0f
        var initialTouchY = 0f
        var lastTapTime = 0L
        var hasMoved = false
        
        overlayView?.setOnTouchListener { _, event ->
            when (event.action) {
                MotionEvent.ACTION_DOWN -> {
                    initialX = params.x
                    initialY = params.y
                    initialTouchX = event.rawX
                    initialTouchY = event.rawY
                    hasMoved = false
                    
                    isInteracting = true
                    true
                }
                
                MotionEvent.ACTION_MOVE -> {
                    val deltaX = (event.rawX - initialTouchX).toInt()
                    val deltaY = (event.rawY - initialTouchY).toInt()
                    
                    if (abs(deltaX) > 10 || abs(deltaY) > 10) {
                        hasMoved = true
                        
                        params.x = initialX + deltaX
                        params.y = initialY + deltaY
                        
                        customX = params.x
                        customY = params.y
                        currentMode = PositionMode.CUSTOM
                        
                        try {
                            windowManager.updateViewLayout(overlayView, params)
                        } catch (e: Exception) {
                            Log.e(TAG, "Error updating layout: ${e.message}")
                        }
                    }
                    true
                }
                
                MotionEvent.ACTION_UP -> {
                    if (!hasMoved) {
                        val currentTime = System.currentTimeMillis()
                        if (currentTime - lastTapTime < 300) {
                            cyclePositionMode(params)
                        } else {
                            toggleAnimation()
                        }
                        lastTapTime = currentTime
                    } else {
                        snapToEdgeIfNeeded(params)
                        saveCurrentState()
                    }
                    
                    scope.launch {
                        delay(2000)
                        isInteracting = false
                    }
                    true
                }
                
                else -> false
            }
        }
    }
    
    private fun cyclePositionMode(params: WindowManager.LayoutParams) {
        currentMode = when (currentMode) {
            PositionMode.TOP_CORNER -> PositionMode.SIDEBAR
            PositionMode.SIDEBAR -> PositionMode.AMBIENT
            PositionMode.AMBIENT -> PositionMode.TOP_CORNER
            PositionMode.CUSTOM -> PositionMode.TOP_CORNER
        }
        
        applyPositionMode(params)
        saveCurrentState()
    }
    
    private fun applyPositionMode(params: WindowManager.LayoutParams, animate: Boolean = true) {
        val displayMetrics = resources.displayMetrics
        val screenWidth = displayMetrics.widthPixels
        val screenHeight = displayMetrics.heightPixels
        
        when (currentMode) {
            PositionMode.TOP_CORNER -> {
                params.gravity = Gravity.TOP or Gravity.END
                params.x = 20
                params.y = 100
                avatarView?.layoutParams = FrameLayout.LayoutParams(320, 320)
            }
            
            PositionMode.SIDEBAR -> {
                params.gravity = Gravity.CENTER_VERTICAL or Gravity.END
                params.x = 10
                params.y = 0
                avatarView?.layoutParams = FrameLayout.LayoutParams(120, 300)
            }
            
            PositionMode.AMBIENT -> {
                params.gravity = Gravity.CENTER
                params.x = 0
                params.y = 0
                avatarView?.layoutParams = FrameLayout.LayoutParams(screenWidth, screenHeight)
            }
            
            PositionMode.CUSTOM -> {
                params.gravity = Gravity.TOP or Gravity.START
                params.x = customX
                params.y = customY
            }
        }
        
        avatarView?.requestLayout()
        
        if (animate) {
            animateLayoutChange(params)
        } else {
            try {
                windowManager.updateViewLayout(overlayView, params)
            } catch (e: Exception) {
                Log.e(TAG, "Error applying position mode: ${e.message}")
            }
        }
    }
    
    private fun animateLayoutChange(params: WindowManager.LayoutParams) {
        overlayView?.animate()
            ?.alpha(0.3f)
            ?.setDuration(200)
            ?.withEndAction {
                try {
                    windowManager.updateViewLayout(overlayView, params)
                    overlayView?.animate()
                        ?.alpha(1f)
                        ?.setDuration(300)
                        ?.start()
                } catch (e: Exception) {
                    Log.e(TAG, "Error in animation: ${e.message}")
                }
            }
            ?.start()
    }
    
    private fun snapToEdgeIfNeeded(params: WindowManager.LayoutParams) {
        val displayMetrics = resources.displayMetrics
        val screenWidth = displayMetrics.widthPixels
        val threshold = 50
        
        if (params.x < threshold) {
            params.x = 0
            try {
                windowManager.updateViewLayout(overlayView, params)
            } catch (e: Exception) {
                Log.e(TAG, "Error snapping to edge: ${e.message}")
            }
        } else if (params.x > screenWidth - threshold) {
            params.gravity = Gravity.TOP or Gravity.END
            params.x = 0
            try {
                windowManager.updateViewLayout(overlayView, params)
            } catch (e: Exception) {
                Log.e(TAG, "Error snapping to edge: ${e.message}")
            }
        }
    }
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        return START_STICKY
    }
    
    override fun onDestroy() {
        super.onDestroy()
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            audioManager.unregisterAudioPlaybackCallback(audioPlaybackCallback)
        }
        
        idleAnimationJob?.cancel()
        monitorJob?.cancel()
        audioMonitorJob?.cancel()
        scope.cancel()
        saveCurrentState()
        
        gifDrawable = null
        
        try {
            if (isViewAttached) {
                overlayView?.let {
                    windowManager.removeView(it)
                }
                isViewAttached = false
            }
        } catch (e: Exception) {
            Log.e(TAG, "Error in onDestroy: ${e.message}")
        }
    }
    
    override fun onBind(intent: Intent?): IBinder? = null
} = Get-Content app\src\main\java\com\overlay\chatgpt\OverlayService.kt -Raw
package com.overlay.chatgpt

import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.app.Service
import android.content.Context
import android.content.Intent
import android.content.SharedPreferences
import android.graphics.PixelFormat
import android.media.AudioManager
import android.media.AudioPlaybackConfiguration
import android.os.Build
import android.os.IBinder
import android.util.Log
import android.view.*
import android.view.animation.AccelerateDecelerateInterpolator
import android.widget.FrameLayout
import android.widget.ImageView
import androidx.core.app.NotificationCompat
import com.bumptech.glide.Glide
import com.bumptech.glide.load.DataSource
import com.bumptech.glide.load.engine.GlideException
import com.bumptech.glide.load.resource.gif.GifDrawable
import com.bumptech.glide.request.RequestListener
import com.bumptech.glide.request.target.Target
import kotlinx.coroutines.*
import kotlin.math.abs

class OverlayService : Service() {
    
    private lateinit var windowManager: WindowManager
    private var overlayView: FrameLayout? = null
    private var avatarView: ImageView? = null
    private lateinit var prefs: SharedPreferences
    private lateinit var audioManager: AudioManager
    
    private val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())
    private var idleAnimationJob: Job? = null
    private var monitorJob: Job? = null
    private var audioMonitorJob: Job? = null
    
    private var gifDrawable: GifDrawable? = null
    
    private val audioPlaybackCallback = object : AudioManager.AudioPlaybackCallback() {
        override fun onPlaybackConfigChanged(configs: MutableList<AudioPlaybackConfiguration>?) {
            super.onPlaybackConfigChanged(configs)
            
            val isPlaying = configs?.isNotEmpty() == true
            
            scope.launch {
                if (isPlaying && !isVoiceModeActive) {
                    isVoiceModeActive = true
                    showAvatar()
                } else if (!isPlaying && isVoiceModeActive) {
                    isVoiceModeActive = false
                    hideAvatar()
                }
            }
        }
    }
    
    enum class PositionMode {
        TOP_CORNER, SIDEBAR, AMBIENT, CUSTOM
    }
    
    private var currentMode = PositionMode.TOP_CORNER
    private var isInteracting = false
    private var isVoiceModeActive = false
    private var customX = 0
    private var customY = 0
    private var isViewAttached = false
    
    companion object {
        private const val TAG = "OverlayService"
        private const val PREFS_NAME = "overlay_prefs"
        private const val KEY_LAST_MODE = "last_mode"
        private const val KEY_CUSTOM_X = "custom_x"
        private const val KEY_CUSTOM_Y = "custom_y"
        private const val KEY_CUSTOM_WIDTH = "custom_width"
        private const val KEY_CUSTOM_HEIGHT = "custom_height"
        private const val CHANNEL_ID = "overlay_service_channel"
        private const val NOTIFICATION_ID = 1
    }
    
    override fun onCreate() {
        super.onCreate()
        
        startForegroundService()
        
        windowManager = getSystemService(Context.WINDOW_SERVICE) as WindowManager
        audioManager = getSystemService(Context.AUDIO_SERVICE) as AudioManager
        prefs = getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE)
        
        restoreLastState()
        createOverlay()
        startIdleAnimation()
        startMonitoring()
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            audioManager.registerAudioPlaybackCallback(audioPlaybackCallback, null)
        }
        
        startAudioMonitoring()
    }
    
    private fun startForegroundService() {
        createNotificationChannel()
        
        val notificationIntent = Intent(this, OverlayControlActivity::class.java)
        val pendingIntent = PendingIntent.getActivity(
            this, 0, notificationIntent,
            PendingIntent.FLAG_IMMUTABLE
        )
        
        val notification = NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("Avatar Overlay")
            .setContentText("Tap to start, tap again to stop")
            .setSmallIcon(android.R.drawable.ic_dialog_info)
            .setContentIntent(pendingIntent)
            .build()
        
        startForeground(NOTIFICATION_ID, notification)
    }
    
    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val serviceChannel = NotificationChannel(
                CHANNEL_ID,
                "Overlay Service",
                NotificationManager.IMPORTANCE_LOW
            )
            val manager = getSystemService(NotificationManager::class.java)
            manager.createNotificationChannel(serviceChannel)
        }
    }
    
    private fun startAudioMonitoring() {
        audioMonitorJob?.cancel()
        audioMonitorJob = scope.launch {
            while (isActive) {
                val isMusicActive = audioManager.isMusicActive
                val hasActivePlayback = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                    audioManager.activePlaybackConfigurations.isNotEmpty()
                } else {
                    false
                }
                
                val isAudioPlaying = isMusicActive || hasActivePlayback
                
                if (isAudioPlaying && !isVoiceModeActive) {
                    isVoiceModeActive = true
                    showAvatar()
                } else if (!isAudioPlaying && isVoiceModeActive) {
                    isVoiceModeActive = false
                    hideAvatar()
                }
                
                delay(300)
            }
        }
    }
    
    private fun showAvatar() {
        avatarView?.animate()
            ?.alpha(0.85f)
            ?.setDuration(200)
            ?.start()
    }
    
    private fun hideAvatar() {
        gifDrawable = null
        avatarView?.setImageDrawable(null)
        
        avatarView?.animate()
            ?.alpha(0.0f)
            ?.setDuration(200)
            ?.start()
    }
    
    private fun toggleAnimation() {
        if (gifDrawable == null) {
            // Load and play GIF
            loadAndPlayGif()
        } else {
            // Stop by completely removing the GIF
            gifDrawable = null
            avatarView?.setImageDrawable(null)
        }
    }
    
    private fun loadAndPlayGif() {
        Glide.with(this)
            .asGif()
            .load(R.raw.avatar_lipsync)
            .listener(object : RequestListener<GifDrawable> {
                override fun onLoadFailed(
                    e: GlideException?,
                    model: Any?,
                    target: Target<GifDrawable>,
                    isFirstResource: Boolean
                ): Boolean {
                    return false
                }
                
                override fun onResourceReady(
                    resource: GifDrawable,
                    model: Any,
                    target: Target<GifDrawable>?,
                    dataSource: DataSource,
                    isFirstResource: Boolean
                ): Boolean {
                    gifDrawable = resource
                    resource.setLoopCount(GifDrawable.LOOP_FOREVER)
                    return false
                }
            })
            .into(avatarView!!)
    }
    
    private fun startMonitoring() {
        monitorJob?.cancel()
        monitorJob = scope.launch {
            while (isActive) {
                delay(500)
                
                if (isViewAttached && overlayView?.windowToken == null) {
                    isViewAttached = false
                    
                    delay(100)
                    try {
                        createOverlay()
                        if (isVoiceModeActive) {
                            showAvatar()
                        }
                    } catch (e: Exception) {
                        Log.e(TAG, "Failed to re-attach: ${e.message}")
                    }
                }
            }
        }
    }
    
    private fun restoreLastState() {
        val savedMode = prefs.getString(KEY_LAST_MODE, PositionMode.TOP_CORNER.name)
        currentMode = try {
            PositionMode.valueOf(savedMode ?: PositionMode.TOP_CORNER.name)
        } catch (e: Exception) {
            PositionMode.TOP_CORNER
        }
        
        if (currentMode == PositionMode.CUSTOM) {
            customX = prefs.getInt(KEY_CUSTOM_X, 50)
            customY = prefs.getInt(KEY_CUSTOM_Y, 100)
        }
    }
    
    private fun saveCurrentState() {
        prefs.edit().apply {
            putString(KEY_LAST_MODE, currentMode.name)
            if (currentMode == PositionMode.CUSTOM) {
                putInt(KEY_CUSTOM_X, customX)
                putInt(KEY_CUSTOM_Y, customY)
                val params = overlayView?.layoutParams as? WindowManager.LayoutParams
                params?.let {
                    putInt(KEY_CUSTOM_WIDTH, avatarView?.width ?: 320)
                    putInt(KEY_CUSTOM_HEIGHT, avatarView?.height ?: 320)
                }
            }
            apply()
        }
    }
    
    private fun createOverlay() {
        try {
            if (overlayView != null && isViewAttached) {
                windowManager.removeView(overlayView)
                isViewAttached = false
            }
        } catch (e: Exception) {
            Log.e(TAG, "Error removing old view: ${e.message}")
        }
        
        overlayView = FrameLayout(this).apply {
            layoutParams = FrameLayout.LayoutParams(
                FrameLayout.LayoutParams.WRAP_CONTENT,
                FrameLayout.LayoutParams.WRAP_CONTENT
            )
        }
        
        val savedWidth = prefs.getInt(KEY_CUSTOM_WIDTH, 320)
        val savedHeight = prefs.getInt(KEY_CUSTOM_HEIGHT, 320)
        
        avatarView = ImageView(this).apply {
            layoutParams = FrameLayout.LayoutParams(savedWidth, savedHeight)
            scaleType = ImageView.ScaleType.FIT_CENTER
            alpha = 0.0f
        }
        
        overlayView?.addView(avatarView)
        
        val params = createWindowParams()
        setupTouchHandling(params)
        
        try {
            windowManager.addView(overlayView, params)
            isViewAttached = true
            
            applyPositionMode(params, animate = false)
        } catch (e: Exception) {
            Log.e(TAG, "Error creating overlay: ${e.message}")
            e.printStackTrace()
            isViewAttached = false
        }
    }
    
    private fun createWindowParams(): WindowManager.LayoutParams {
        val layoutType = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
        } else {
            @Suppress("DEPRECATION")
            WindowManager.LayoutParams.TYPE_PHONE
        }
        
        return WindowManager.LayoutParams(
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.WRAP_CONTENT,
            layoutType,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
                    WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL or
                    WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH or
                    WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS,
            PixelFormat.TRANSLUCENT
        ).apply {
            gravity = Gravity.TOP or Gravity.START
            x = if (currentMode == PositionMode.CUSTOM) customX else 50
            y = if (currentMode == PositionMode.CUSTOM) customY else 100
        }
    }
    
    private fun startIdleAnimation() {
        idleAnimationJob?.cancel()
        idleAnimationJob = scope.launch {
            while (isActive) {
                delay(3000)
                
                if (!isInteracting && isViewAttached && gifDrawable == null && isVoiceModeActive) {
                    avatarView?.animate()
                        ?.alpha(0.7f)
                        ?.setDuration(1500)
                        ?.setInterpolator(AccelerateDecelerateInterpolator())
                        ?.withEndAction {
                            avatarView?.animate()
                                ?.alpha(0.85f)
                                ?.setDuration(1500)
                                ?.setInterpolator(AccelerateDecelerateInterpolator())
                                ?.start()
                        }
                        ?.start()
                }
            }
        }
    }
    
    private fun setupTouchHandling(params: WindowManager.LayoutParams) {
        var initialX = 0
        var initialY = 0
        var initialTouchX = 0f
        var initialTouchY = 0f
        var lastTapTime = 0L
        var hasMoved = false
        
        overlayView?.setOnTouchListener { _, event ->
            when (event.action) {
                MotionEvent.ACTION_DOWN -> {
                    initialX = params.x
                    initialY = params.y
                    initialTouchX = event.rawX
                    initialTouchY = event.rawY
                    hasMoved = false
                    
                    isInteracting = true
                    true
                }
                
                MotionEvent.ACTION_MOVE -> {
                    val deltaX = (event.rawX - initialTouchX).toInt()
                    val deltaY = (event.rawY - initialTouchY).toInt()
                    
                    if (abs(deltaX) > 10 || abs(deltaY) > 10) {
                        hasMoved = true
                        
                        params.x = initialX + deltaX
                        params.y = initialY + deltaY
                        
                        customX = params.x
                        customY = params.y
                        currentMode = PositionMode.CUSTOM
                        
                        try {
                            windowManager.updateViewLayout(overlayView, params)
                        } catch (e: Exception) {
                            Log.e(TAG, "Error updating layout: ${e.message}")
                        }
                    }
                    true
                }
                
                MotionEvent.ACTION_UP -> {
                    if (!hasMoved) {
                        val currentTime = System.currentTimeMillis()
                        if (currentTime - lastTapTime < 300) {
                            cyclePositionMode(params)
                        } else {
                            toggleAnimation()
                        }
                        lastTapTime = currentTime
                    } else {
                        snapToEdgeIfNeeded(params)
                        saveCurrentState()
                    }
                    
                    scope.launch {
                        delay(2000)
                        isInteracting = false
                    }
                    true
                }
                
                else -> false
            }
        }
    }
    
    private fun cyclePositionMode(params: WindowManager.LayoutParams) {
        currentMode = when (currentMode) {
            PositionMode.TOP_CORNER -> PositionMode.SIDEBAR
            PositionMode.SIDEBAR -> PositionMode.AMBIENT
            PositionMode.AMBIENT -> PositionMode.TOP_CORNER
            PositionMode.CUSTOM -> PositionMode.TOP_CORNER
        }
        
        applyPositionMode(params)
        saveCurrentState()
    }
    
    private fun applyPositionMode(params: WindowManager.LayoutParams, animate: Boolean = true) {
        val displayMetrics = resources.displayMetrics
        val screenWidth = displayMetrics.widthPixels
        val screenHeight = displayMetrics.heightPixels
        
        when (currentMode) {
            PositionMode.TOP_CORNER -> {
                params.gravity = Gravity.TOP or Gravity.END
                params.x = 20
                params.y = 100
                avatarView?.layoutParams = FrameLayout.LayoutParams(320, 320)
            }
            
            PositionMode.SIDEBAR -> {
                params.gravity = Gravity.CENTER_VERTICAL or Gravity.END
                params.x = 10
                params.y = 0
                avatarView?.layoutParams = FrameLayout.LayoutParams(120, 300)
            }
            
            PositionMode.AMBIENT -> {
                params.gravity = Gravity.CENTER
                params.x = 0
                params.y = 0
                avatarView?.layoutParams = FrameLayout.LayoutParams(screenWidth, screenHeight)
            }
            
            PositionMode.CUSTOM -> {
                params.gravity = Gravity.TOP or Gravity.START
                params.x = customX
                params.y = customY
            }
        }
        
        avatarView?.requestLayout()
        
        if (animate) {
            animateLayoutChange(params)
        } else {
            try {
                windowManager.updateViewLayout(overlayView, params)
            } catch (e: Exception) {
                Log.e(TAG, "Error applying position mode: ${e.message}")
            }
        }
    }
    
    private fun animateLayoutChange(params: WindowManager.LayoutParams) {
        overlayView?.animate()
            ?.alpha(0.3f)
            ?.setDuration(200)
            ?.withEndAction {
                try {
                    windowManager.updateViewLayout(overlayView, params)
                    overlayView?.animate()
                        ?.alpha(1f)
                        ?.setDuration(300)
                        ?.start()
                } catch (e: Exception) {
                    Log.e(TAG, "Error in animation: ${e.message}")
                }
            }
            ?.start()
    }
    
    private fun snapToEdgeIfNeeded(params: WindowManager.LayoutParams) {
        val displayMetrics = resources.displayMetrics
        val screenWidth = displayMetrics.widthPixels
        val threshold = 50
        
        if (params.x < threshold) {
            params.x = 0
            try {
                windowManager.updateViewLayout(overlayView, params)
            } catch (e: Exception) {
                Log.e(TAG, "Error snapping to edge: ${e.message}")
            }
        } else if (params.x > screenWidth - threshold) {
            params.gravity = Gravity.TOP or Gravity.END
            params.x = 0
            try {
                windowManager.updateViewLayout(overlayView, params)
            } catch (e: Exception) {
                Log.e(TAG, "Error snapping to edge: ${e.message}")
            }
        }
    }
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        return START_STICKY
    }
    
    override fun onDestroy() {
        super.onDestroy()
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            audioManager.unregisterAudioPlaybackCallback(audioPlaybackCallback)
        }
        
        idleAnimationJob?.cancel()
        monitorJob?.cancel()
        audioMonitorJob?.cancel()
        scope.cancel()
        saveCurrentState()
        
        gifDrawable = null
        
        try {
            if (isViewAttached) {
                overlayView?.let {
                    windowManager.removeView(it)
                }
                isViewAttached = false
            }
        } catch (e: Exception) {
            Log.e(TAG, "Error in onDestroy: ${e.message}")
        }
    }
    
    override fun onBind(intent: Intent?): IBinder? = null
} = package com.overlay.chatgpt

import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.app.Service
import android.content.Context
import android.content.Intent
import android.content.SharedPreferences
import android.graphics.PixelFormat
import android.media.AudioManager
import android.media.AudioPlaybackConfiguration
import android.os.Build
import android.os.IBinder
import android.util.Log
import android.view.*
import android.view.animation.AccelerateDecelerateInterpolator
import android.widget.FrameLayout
import android.widget.ImageView
import androidx.core.app.NotificationCompat
import com.bumptech.glide.Glide
import com.bumptech.glide.load.DataSource
import com.bumptech.glide.load.engine.GlideException
import com.bumptech.glide.load.resource.gif.GifDrawable
import com.bumptech.glide.request.RequestListener
import com.bumptech.glide.request.target.Target
import kotlinx.coroutines.*
import kotlin.math.abs

class OverlayService : Service() {
    
    private lateinit var windowManager: WindowManager
    private var overlayView: FrameLayout? = null
    private var avatarView: ImageView? = null
    private lateinit var prefs: SharedPreferences
    private lateinit var audioManager: AudioManager
    
    private val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())
    private var idleAnimationJob: Job? = null
    private var monitorJob: Job? = null
    private var audioMonitorJob: Job? = null
    
    private var gifDrawable: GifDrawable? = null
    
    private val audioPlaybackCallback = object : AudioManager.AudioPlaybackCallback() {
        override fun onPlaybackConfigChanged(configs: MutableList<AudioPlaybackConfiguration>?) {
            super.onPlaybackConfigChanged(configs)
            
            val isPlaying = configs?.isNotEmpty() == true
            
            scope.launch {
                if (isPlaying && !isVoiceModeActive) {
                    isVoiceModeActive = true
                    showAvatar()
                } else if (!isPlaying && isVoiceModeActive) {
                    isVoiceModeActive = false
                    hideAvatar()
                }
            }
        }
    }
    
    enum class PositionMode {
        TOP_CORNER, SIDEBAR, AMBIENT, CUSTOM
    }
    
    private var currentMode = PositionMode.TOP_CORNER
    private var isInteracting = false
    private var isVoiceModeActive = false
    private var customX = 0
    private var customY = 0
    private var isViewAttached = false
    
    companion object {
        private const val TAG = "OverlayService"
        private const val PREFS_NAME = "overlay_prefs"
        private const val KEY_LAST_MODE = "last_mode"
        private const val KEY_CUSTOM_X = "custom_x"
        private const val KEY_CUSTOM_Y = "custom_y"
        private const val KEY_CUSTOM_WIDTH = "custom_width"
        private const val KEY_CUSTOM_HEIGHT = "custom_height"
        private const val CHANNEL_ID = "overlay_service_channel"
        private const val NOTIFICATION_ID = 1
    }
    
    override fun onCreate() {
        super.onCreate()
        
        startForegroundService()
        
        windowManager = getSystemService(Context.WINDOW_SERVICE) as WindowManager
        audioManager = getSystemService(Context.AUDIO_SERVICE) as AudioManager
        prefs = getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE)
        
        restoreLastState()
        createOverlay()
        startIdleAnimation()
        startMonitoring()
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            audioManager.registerAudioPlaybackCallback(audioPlaybackCallback, null)
        }
        
        startAudioMonitoring()
    }
    
    private fun startForegroundService() {
        createNotificationChannel()
        
        val notificationIntent = Intent(this, OverlayControlActivity::class.java)
        val pendingIntent = PendingIntent.getActivity(
            this, 0, notificationIntent,
            PendingIntent.FLAG_IMMUTABLE
        )
        
        val notification = NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("Avatar Overlay")
            .setContentText("Tap to start, tap again to stop")
            .setSmallIcon(android.R.drawable.ic_dialog_info)
            .setContentIntent(pendingIntent)
            .build()
        
        startForeground(NOTIFICATION_ID, notification)
    }
    
    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val serviceChannel = NotificationChannel(
                CHANNEL_ID,
                "Overlay Service",
                NotificationManager.IMPORTANCE_LOW
            )
            val manager = getSystemService(NotificationManager::class.java)
            manager.createNotificationChannel(serviceChannel)
        }
    }
    
    private fun startAudioMonitoring() {
        audioMonitorJob?.cancel()
        audioMonitorJob = scope.launch {
            while (isActive) {
                val isMusicActive = audioManager.isMusicActive
                val hasActivePlayback = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                    audioManager.activePlaybackConfigurations.isNotEmpty()
                } else {
                    false
                }
                
                val isAudioPlaying = isMusicActive || hasActivePlayback
                
                if (isAudioPlaying && !isVoiceModeActive) {
                    isVoiceModeActive = true
                    showAvatar()
                } else if (!isAudioPlaying && isVoiceModeActive) {
                    isVoiceModeActive = false
                    hideAvatar()
                }
                
                delay(300)
            }
        }
    }
    
    private fun showAvatar() {
        avatarView?.animate()
            ?.alpha(0.85f)
            ?.setDuration(200)
            ?.start()
    }
    
    private fun hideAvatar() {
        gifDrawable = null
        avatarView?.setImageDrawable(null)
        
        avatarView?.animate()
            ?.alpha(0.0f)
            ?.setDuration(200)
            ?.start()
    }
    
    private fun toggleAnimation() {
        if (gifDrawable == null) {
            // Load and play GIF
            loadAndPlayGif()
        } else {
            // Stop by completely removing the GIF
            gifDrawable = null
            avatarView?.setImageDrawable(null)
        }
    }
    
    private fun loadAndPlayGif() {
        Glide.with(this)
            .asGif()
            .load(R.raw.avatar_lipsync)
            .listener(object : RequestListener<GifDrawable> {
                override fun onLoadFailed(
                    e: GlideException?,
                    model: Any?,
                    target: Target<GifDrawable>,
                    isFirstResource: Boolean
                ): Boolean {
                    return false
                }
                
                override fun onResourceReady(
                    resource: GifDrawable,
                    model: Any,
                    target: Target<GifDrawable>?,
                    dataSource: DataSource,
                    isFirstResource: Boolean
                ): Boolean {
                    gifDrawable = resource
                    resource.setLoopCount(GifDrawable.LOOP_FOREVER)
                    return false
                }
            })
            .into(avatarView!!)
    }
    
    private fun startMonitoring() {
        monitorJob?.cancel()
        monitorJob = scope.launch {
            while (isActive) {
                delay(500)
                
                if (isViewAttached && overlayView?.windowToken == null) {
                    isViewAttached = false
                    
                    delay(100)
                    try {
                        createOverlay()
                        if (isVoiceModeActive) {
                            showAvatar()
                        }
                    } catch (e: Exception) {
                        Log.e(TAG, "Failed to re-attach: ${e.message}")
                    }
                }
            }
        }
    }
    
    private fun restoreLastState() {
        val savedMode = prefs.getString(KEY_LAST_MODE, PositionMode.TOP_CORNER.name)
        currentMode = try {
            PositionMode.valueOf(savedMode ?: PositionMode.TOP_CORNER.name)
        } catch (e: Exception) {
            PositionMode.TOP_CORNER
        }
        
        if (currentMode == PositionMode.CUSTOM) {
            customX = prefs.getInt(KEY_CUSTOM_X, 50)
            customY = prefs.getInt(KEY_CUSTOM_Y, 100)
        }
    }
    
    private fun saveCurrentState() {
        prefs.edit().apply {
            putString(KEY_LAST_MODE, currentMode.name)
            if (currentMode == PositionMode.CUSTOM) {
                putInt(KEY_CUSTOM_X, customX)
                putInt(KEY_CUSTOM_Y, customY)
                val params = overlayView?.layoutParams as? WindowManager.LayoutParams
                params?.let {
                    putInt(KEY_CUSTOM_WIDTH, avatarView?.width ?: 320)
                    putInt(KEY_CUSTOM_HEIGHT, avatarView?.height ?: 320)
                }
            }
            apply()
        }
    }
    
    private fun createOverlay() {
        try {
            if (overlayView != null && isViewAttached) {
                windowManager.removeView(overlayView)
                isViewAttached = false
            }
        } catch (e: Exception) {
            Log.e(TAG, "Error removing old view: ${e.message}")
        }
        
        overlayView = FrameLayout(this).apply {
            layoutParams = FrameLayout.LayoutParams(
                FrameLayout.LayoutParams.WRAP_CONTENT,
                FrameLayout.LayoutParams.WRAP_CONTENT
            )
        }
        
        val savedWidth = prefs.getInt(KEY_CUSTOM_WIDTH, 320)
        val savedHeight = prefs.getInt(KEY_CUSTOM_HEIGHT, 320)
        
        avatarView = ImageView(this).apply {
            layoutParams = FrameLayout.LayoutParams(savedWidth, savedHeight)
            scaleType = ImageView.ScaleType.FIT_CENTER
            alpha = 0.0f
        }
        
        overlayView?.addView(avatarView)
        
        val params = createWindowParams()
        setupTouchHandling(params)
        
        try {
            windowManager.addView(overlayView, params)
            isViewAttached = true
            
            applyPositionMode(params, animate = false)
        } catch (e: Exception) {
            Log.e(TAG, "Error creating overlay: ${e.message}")
            e.printStackTrace()
            isViewAttached = false
        }
    }
    
    private fun createWindowParams(): WindowManager.LayoutParams {
        val layoutType = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
        } else {
            @Suppress("DEPRECATION")
            WindowManager.LayoutParams.TYPE_PHONE
        }
        
        return WindowManager.LayoutParams(
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.WRAP_CONTENT,
            layoutType,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
                    WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL or
                    WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH or
                    WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS,
            PixelFormat.TRANSLUCENT
        ).apply {
            gravity = Gravity.TOP or Gravity.START
            x = if (currentMode == PositionMode.CUSTOM) customX else 50
            y = if (currentMode == PositionMode.CUSTOM) customY else 100
        }
    }
    
    private fun startIdleAnimation() {
        idleAnimationJob?.cancel()
        idleAnimationJob = scope.launch {
            while (isActive) {
                delay(3000)
                
                if (!isInteracting && isViewAttached && gifDrawable == null && isVoiceModeActive) {
                    avatarView?.animate()
                        ?.alpha(0.7f)
                        ?.setDuration(1500)
                        ?.setInterpolator(AccelerateDecelerateInterpolator())
                        ?.withEndAction {
                            avatarView?.animate()
                                ?.alpha(0.85f)
                                ?.setDuration(1500)
                                ?.setInterpolator(AccelerateDecelerateInterpolator())
                                ?.start()
                        }
                        ?.start()
                }
            }
        }
    }
    
    private fun setupTouchHandling(params: WindowManager.LayoutParams) {
        var initialX = 0
        var initialY = 0
        var initialTouchX = 0f
        var initialTouchY = 0f
        var lastTapTime = 0L
        var hasMoved = false
        
        overlayView?.setOnTouchListener { _, event ->
            when (event.action) {
                MotionEvent.ACTION_DOWN -> {
                    initialX = params.x
                    initialY = params.y
                    initialTouchX = event.rawX
                    initialTouchY = event.rawY
                    hasMoved = false
                    
                    isInteracting = true
                    true
                }
                
                MotionEvent.ACTION_MOVE -> {
                    val deltaX = (event.rawX - initialTouchX).toInt()
                    val deltaY = (event.rawY - initialTouchY).toInt()
                    
                    if (abs(deltaX) > 10 || abs(deltaY) > 10) {
                        hasMoved = true
                        
                        params.x = initialX + deltaX
                        params.y = initialY + deltaY
                        
                        customX = params.x
                        customY = params.y
                        currentMode = PositionMode.CUSTOM
                        
                        try {
                            windowManager.updateViewLayout(overlayView, params)
                        } catch (e: Exception) {
                            Log.e(TAG, "Error updating layout: ${e.message}")
                        }
                    }
                    true
                }
                
                MotionEvent.ACTION_UP -> {
                    if (!hasMoved) {
                        val currentTime = System.currentTimeMillis()
                        if (currentTime - lastTapTime < 300) {
                            cyclePositionMode(params)
                        } else {
                            toggleAnimation()
                        }
                        lastTapTime = currentTime
                    } else {
                        snapToEdgeIfNeeded(params)
                        saveCurrentState()
                    }
                    
                    scope.launch {
                        delay(2000)
                        isInteracting = false
                    }
                    true
                }
                
                else -> false
            }
        }
    }
    
    private fun cyclePositionMode(params: WindowManager.LayoutParams) {
        currentMode = when (currentMode) {
            PositionMode.TOP_CORNER -> PositionMode.SIDEBAR
            PositionMode.SIDEBAR -> PositionMode.AMBIENT
            PositionMode.AMBIENT -> PositionMode.TOP_CORNER
            PositionMode.CUSTOM -> PositionMode.TOP_CORNER
        }
        
        applyPositionMode(params)
        saveCurrentState()
    }
    
    private fun applyPositionMode(params: WindowManager.LayoutParams, animate: Boolean = true) {
        val displayMetrics = resources.displayMetrics
        val screenWidth = displayMetrics.widthPixels
        val screenHeight = displayMetrics.heightPixels
        
        when (currentMode) {
            PositionMode.TOP_CORNER -> {
                params.gravity = Gravity.TOP or Gravity.END
                params.x = 20
                params.y = 100
                avatarView?.layoutParams = FrameLayout.LayoutParams(320, 320)
            }
            
            PositionMode.SIDEBAR -> {
                params.gravity = Gravity.CENTER_VERTICAL or Gravity.END
                params.x = 10
                params.y = 0
                avatarView?.layoutParams = FrameLayout.LayoutParams(120, 300)
            }
            
            PositionMode.AMBIENT -> {
                params.gravity = Gravity.CENTER
                params.x = 0
                params.y = 0
                avatarView?.layoutParams = FrameLayout.LayoutParams(screenWidth, screenHeight)
            }
            
            PositionMode.CUSTOM -> {
                params.gravity = Gravity.TOP or Gravity.START
                params.x = customX
                params.y = customY
            }
        }
        
        avatarView?.requestLayout()
        
        if (animate) {
            animateLayoutChange(params)
        } else {
            try {
                windowManager.updateViewLayout(overlayView, params)
            } catch (e: Exception) {
                Log.e(TAG, "Error applying position mode: ${e.message}")
            }
        }
    }
    
    private fun animateLayoutChange(params: WindowManager.LayoutParams) {
        overlayView?.animate()
            ?.alpha(0.3f)
            ?.setDuration(200)
            ?.withEndAction {
                try {
                    windowManager.updateViewLayout(overlayView, params)
                    overlayView?.animate()
                        ?.alpha(1f)
                        ?.setDuration(300)
                        ?.start()
                } catch (e: Exception) {
                    Log.e(TAG, "Error in animation: ${e.message}")
                }
            }
            ?.start()
    }
    
    private fun snapToEdgeIfNeeded(params: WindowManager.LayoutParams) {
        val displayMetrics = resources.displayMetrics
        val screenWidth = displayMetrics.widthPixels
        val threshold = 50
        
        if (params.x < threshold) {
            params.x = 0
            try {
                windowManager.updateViewLayout(overlayView, params)
            } catch (e: Exception) {
                Log.e(TAG, "Error snapping to edge: ${e.message}")
            }
        } else if (params.x > screenWidth - threshold) {
            params.gravity = Gravity.TOP or Gravity.END
            params.x = 0
            try {
                windowManager.updateViewLayout(overlayView, params)
            } catch (e: Exception) {
                Log.e(TAG, "Error snapping to edge: ${e.message}")
            }
        }
    }
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        return START_STICKY
    }
    
    override fun onDestroy() {
        super.onDestroy()
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            audioManager.unregisterAudioPlaybackCallback(audioPlaybackCallback)
        }
        
        idleAnimationJob?.cancel()
        monitorJob?.cancel()
        audioMonitorJob?.cancel()
        scope.cancel()
        saveCurrentState()
        
        gifDrawable = null
        
        try {
            if (isViewAttached) {
                overlayView?.let {
                    windowManager.removeView(it)
                }
                isViewAttached = false
            }
        } catch (e: Exception) {
            Log.e(TAG, "Error in onDestroy: ${e.message}")
        }
    }
    
    override fun onBind(intent: Intent?): IBinder? = null
} -replace '\s+setBackgroundColor\(0x40FFFFFF\)', ''
Set-Content app\src\main\java\com\overlay\chatgpt\OverlayService.kt -Value package com.overlay.chatgpt

import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.app.Service
import android.content.Context
import android.content.Intent
import android.content.SharedPreferences
import android.graphics.PixelFormat
import android.media.AudioManager
import android.media.AudioPlaybackConfiguration
import android.os.Build
import android.os.IBinder
import android.util.Log
import android.view.*
import android.view.animation.AccelerateDecelerateInterpolator
import android.widget.FrameLayout
import android.widget.ImageView
import androidx.core.app.NotificationCompat
import com.bumptech.glide.Glide
import com.bumptech.glide.load.DataSource
import com.bumptech.glide.load.engine.GlideException
import com.bumptech.glide.load.resource.gif.GifDrawable
import com.bumptech.glide.request.RequestListener
import com.bumptech.glide.request.target.Target
import kotlinx.coroutines.*
import kotlin.math.abs

class OverlayService : Service() {
    
    private lateinit var windowManager: WindowManager
    private var overlayView: FrameLayout? = null
    private var avatarView: ImageView? = null
    private lateinit var prefs: SharedPreferences
    private lateinit var audioManager: AudioManager
    
    private val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())
    private var idleAnimationJob: Job? = null
    private var monitorJob: Job? = null
    private var audioMonitorJob: Job? = null
    
    private var gifDrawable: GifDrawable? = null
    
    private val audioPlaybackCallback = object : AudioManager.AudioPlaybackCallback() {
        override fun onPlaybackConfigChanged(configs: MutableList<AudioPlaybackConfiguration>?) {
            super.onPlaybackConfigChanged(configs)
            
            val isPlaying = configs?.isNotEmpty() == true
            
            scope.launch {
                if (isPlaying && !isVoiceModeActive) {
                    isVoiceModeActive = true
                    showAvatar()
                } else if (!isPlaying && isVoiceModeActive) {
                    isVoiceModeActive = false
                    hideAvatar()
                }
            }
        }
    }
    
    enum class PositionMode {
        TOP_CORNER, SIDEBAR, AMBIENT, CUSTOM
    }
    
    private var currentMode = PositionMode.TOP_CORNER
    private var isInteracting = false
    private var isVoiceModeActive = false
    private var customX = 0
    private var customY = 0
    private var isViewAttached = false
    
    companion object {
        private const val TAG = "OverlayService"
        private const val PREFS_NAME = "overlay_prefs"
        private const val KEY_LAST_MODE = "last_mode"
        private const val KEY_CUSTOM_X = "custom_x"
        private const val KEY_CUSTOM_Y = "custom_y"
        private const val KEY_CUSTOM_WIDTH = "custom_width"
        private const val KEY_CUSTOM_HEIGHT = "custom_height"
        private const val CHANNEL_ID = "overlay_service_channel"
        private const val NOTIFICATION_ID = 1
    }
    
    override fun onCreate() {
        super.onCreate()
        
        startForegroundService()
        
        windowManager = getSystemService(Context.WINDOW_SERVICE) as WindowManager
        audioManager = getSystemService(Context.AUDIO_SERVICE) as AudioManager
        prefs = getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE)
        
        restoreLastState()
        createOverlay()
        startIdleAnimation()
        startMonitoring()
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            audioManager.registerAudioPlaybackCallback(audioPlaybackCallback, null)
        }
        
        startAudioMonitoring()
    }
    
    private fun startForegroundService() {
        createNotificationChannel()
        
        val notificationIntent = Intent(this, OverlayControlActivity::class.java)
        val pendingIntent = PendingIntent.getActivity(
            this, 0, notificationIntent,
            PendingIntent.FLAG_IMMUTABLE
        )
        
        val notification = NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("Avatar Overlay")
            .setContentText("Tap to start, tap again to stop")
            .setSmallIcon(android.R.drawable.ic_dialog_info)
            .setContentIntent(pendingIntent)
            .build()
        
        startForeground(NOTIFICATION_ID, notification)
    }
    
    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val serviceChannel = NotificationChannel(
                CHANNEL_ID,
                "Overlay Service",
                NotificationManager.IMPORTANCE_LOW
            )
            val manager = getSystemService(NotificationManager::class.java)
            manager.createNotificationChannel(serviceChannel)
        }
    }
    
    private fun startAudioMonitoring() {
        audioMonitorJob?.cancel()
        audioMonitorJob = scope.launch {
            while (isActive) {
                val isMusicActive = audioManager.isMusicActive
                val hasActivePlayback = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                    audioManager.activePlaybackConfigurations.isNotEmpty()
                } else {
                    false
                }
                
                val isAudioPlaying = isMusicActive || hasActivePlayback
                
                if (isAudioPlaying && !isVoiceModeActive) {
                    isVoiceModeActive = true
                    showAvatar()
                } else if (!isAudioPlaying && isVoiceModeActive) {
                    isVoiceModeActive = false
                    hideAvatar()
                }
                
                delay(300)
            }
        }
    }
    
    private fun showAvatar() {
        avatarView?.animate()
            ?.alpha(0.85f)
            ?.setDuration(200)
            ?.start()
    }
    
    private fun hideAvatar() {
        gifDrawable = null
        avatarView?.setImageDrawable(null)
        
        avatarView?.animate()
            ?.alpha(0.0f)
            ?.setDuration(200)
            ?.start()
    }
    
    private fun toggleAnimation() {
        if (gifDrawable == null) {
            // Load and play GIF
            loadAndPlayGif()
        } else {
            // Stop by completely removing the GIF
            gifDrawable = null
            avatarView?.setImageDrawable(null)
        }
    }
    
    private fun loadAndPlayGif() {
        Glide.with(this)
            .asGif()
            .load(R.raw.avatar_lipsync)
            .listener(object : RequestListener<GifDrawable> {
                override fun onLoadFailed(
                    e: GlideException?,
                    model: Any?,
                    target: Target<GifDrawable>,
                    isFirstResource: Boolean
                ): Boolean {
                    return false
                }
                
                override fun onResourceReady(
                    resource: GifDrawable,
                    model: Any,
                    target: Target<GifDrawable>?,
                    dataSource: DataSource,
                    isFirstResource: Boolean
                ): Boolean {
                    gifDrawable = resource
                    resource.setLoopCount(GifDrawable.LOOP_FOREVER)
                    return false
                }
            })
            .into(avatarView!!)
    }
    
    private fun startMonitoring() {
        monitorJob?.cancel()
        monitorJob = scope.launch {
            while (isActive) {
                delay(500)
                
                if (isViewAttached && overlayView?.windowToken == null) {
                    isViewAttached = false
                    
                    delay(100)
                    try {
                        createOverlay()
                        if (isVoiceModeActive) {
                            showAvatar()
                        }
                    } catch (e: Exception) {
                        Log.e(TAG, "Failed to re-attach: ${e.message}")
                    }
                }
            }
        }
    }
    
    private fun restoreLastState() {
        val savedMode = prefs.getString(KEY_LAST_MODE, PositionMode.TOP_CORNER.name)
        currentMode = try {
            PositionMode.valueOf(savedMode ?: PositionMode.TOP_CORNER.name)
        } catch (e: Exception) {
            PositionMode.TOP_CORNER
        }
        
        if (currentMode == PositionMode.CUSTOM) {
            customX = prefs.getInt(KEY_CUSTOM_X, 50)
            customY = prefs.getInt(KEY_CUSTOM_Y, 100)
        }
    }
    
    private fun saveCurrentState() {
        prefs.edit().apply {
            putString(KEY_LAST_MODE, currentMode.name)
            if (currentMode == PositionMode.CUSTOM) {
                putInt(KEY_CUSTOM_X, customX)
                putInt(KEY_CUSTOM_Y, customY)
                val params = overlayView?.layoutParams as? WindowManager.LayoutParams
                params?.let {
                    putInt(KEY_CUSTOM_WIDTH, avatarView?.width ?: 320)
                    putInt(KEY_CUSTOM_HEIGHT, avatarView?.height ?: 320)
                }
            }
            apply()
        }
    }
    
    private fun createOverlay() {
        try {
            if (overlayView != null && isViewAttached) {
                windowManager.removeView(overlayView)
                isViewAttached = false
            }
        } catch (e: Exception) {
            Log.e(TAG, "Error removing old view: ${e.message}")
        }
        
        overlayView = FrameLayout(this).apply {
            layoutParams = FrameLayout.LayoutParams(
                FrameLayout.LayoutParams.WRAP_CONTENT,
                FrameLayout.LayoutParams.WRAP_CONTENT
            )
        }
        
        val savedWidth = prefs.getInt(KEY_CUSTOM_WIDTH, 320)
        val savedHeight = prefs.getInt(KEY_CUSTOM_HEIGHT, 320)
        
        avatarView = ImageView(this).apply {
            layoutParams = FrameLayout.LayoutParams(savedWidth, savedHeight)
            scaleType = ImageView.ScaleType.FIT_CENTER
            alpha = 0.0f
        }
        
        overlayView?.addView(avatarView)
        
        val params = createWindowParams()
        setupTouchHandling(params)
        
        try {
            windowManager.addView(overlayView, params)
            isViewAttached = true
            
            applyPositionMode(params, animate = false)
        } catch (e: Exception) {
            Log.e(TAG, "Error creating overlay: ${e.message}")
            e.printStackTrace()
            isViewAttached = false
        }
    }
    
    private fun createWindowParams(): WindowManager.LayoutParams {
        val layoutType = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY
        } else {
            @Suppress("DEPRECATION")
            WindowManager.LayoutParams.TYPE_PHONE
        }
        
        return WindowManager.LayoutParams(
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.WRAP_CONTENT,
            layoutType,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
                    WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL or
                    WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH or
                    WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS,
            PixelFormat.TRANSLUCENT
        ).apply {
            gravity = Gravity.TOP or Gravity.START
            x = if (currentMode == PositionMode.CUSTOM) customX else 50
            y = if (currentMode == PositionMode.CUSTOM) customY else 100
        }
    }
    
    private fun startIdleAnimation() {
        idleAnimationJob?.cancel()
        idleAnimationJob = scope.launch {
            while (isActive) {
                delay(3000)
                
                if (!isInteracting && isViewAttached && gifDrawable == null && isVoiceModeActive) {
                    avatarView?.animate()
                        ?.alpha(0.7f)
                        ?.setDuration(1500)
                        ?.setInterpolator(AccelerateDecelerateInterpolator())
                        ?.withEndAction {
                            avatarView?.animate()
                                ?.alpha(0.85f)
                                ?.setDuration(1500)
                                ?.setInterpolator(AccelerateDecelerateInterpolator())
                                ?.start()
                        }
                        ?.start()
                }
            }
        }
    }
    
    private fun setupTouchHandling(params: WindowManager.LayoutParams) {
        var initialX = 0
        var initialY = 0
        var initialTouchX = 0f
        var initialTouchY = 0f
        var lastTapTime = 0L
        var hasMoved = false
        
        overlayView?.setOnTouchListener { _, event ->
            when (event.action) {
                MotionEvent.ACTION_DOWN -> {
                    initialX = params.x
                    initialY = params.y
                    initialTouchX = event.rawX
                    initialTouchY = event.rawY
                    hasMoved = false
                    
                    isInteracting = true
                    true
                }
                
                MotionEvent.ACTION_MOVE -> {
                    val deltaX = (event.rawX - initialTouchX).toInt()
                    val deltaY = (event.rawY - initialTouchY).toInt()
                    
                    if (abs(deltaX) > 10 || abs(deltaY) > 10) {
                        hasMoved = true
                        
                        params.x = initialX + deltaX
                        params.y = initialY + deltaY
                        
                        customX = params.x
                        customY = params.y
                        currentMode = PositionMode.CUSTOM
                        
                        try {
                            windowManager.updateViewLayout(overlayView, params)
                        } catch (e: Exception) {
                            Log.e(TAG, "Error updating layout: ${e.message}")
                        }
                    }
                    true
                }
                
                MotionEvent.ACTION_UP -> {
                    if (!hasMoved) {
                        val currentTime = System.currentTimeMillis()
                        if (currentTime - lastTapTime < 300) {
                            cyclePositionMode(params)
                        } else {
                            toggleAnimation()
                        }
                        lastTapTime = currentTime
                    } else {
                        snapToEdgeIfNeeded(params)
                        saveCurrentState()
                    }
                    
                    scope.launch {
                        delay(2000)
                        isInteracting = false
                    }
                    true
                }
                
                else -> false
            }
        }
    }
    
    private fun cyclePositionMode(params: WindowManager.LayoutParams) {
        currentMode = when (currentMode) {
            PositionMode.TOP_CORNER -> PositionMode.SIDEBAR
            PositionMode.SIDEBAR -> PositionMode.AMBIENT
            PositionMode.AMBIENT -> PositionMode.TOP_CORNER
            PositionMode.CUSTOM -> PositionMode.TOP_CORNER
        }
        
        applyPositionMode(params)
        saveCurrentState()
    }
    
    private fun applyPositionMode(params: WindowManager.LayoutParams, animate: Boolean = true) {
        val displayMetrics = resources.displayMetrics
        val screenWidth = displayMetrics.widthPixels
        val screenHeight = displayMetrics.heightPixels
        
        when (currentMode) {
            PositionMode.TOP_CORNER -> {
                params.gravity = Gravity.TOP or Gravity.END
                params.x = 20
                params.y = 100
                avatarView?.layoutParams = FrameLayout.LayoutParams(320, 320)
            }
            
            PositionMode.SIDEBAR -> {
                params.gravity = Gravity.CENTER_VERTICAL or Gravity.END
                params.x = 10
                params.y = 0
                avatarView?.layoutParams = FrameLayout.LayoutParams(120, 300)
            }
            
            PositionMode.AMBIENT -> {
                params.gravity = Gravity.CENTER
                params.x = 0
                params.y = 0
                avatarView?.layoutParams = FrameLayout.LayoutParams(screenWidth, screenHeight)
            }
            
            PositionMode.CUSTOM -> {
                params.gravity = Gravity.TOP or Gravity.START
                params.x = customX
                params.y = customY
            }
        }
        
        avatarView?.requestLayout()
        
        if (animate) {
            animateLayoutChange(params)
        } else {
            try {
                windowManager.updateViewLayout(overlayView, params)
            } catch (e: Exception) {
                Log.e(TAG, "Error applying position mode: ${e.message}")
            }
        }
    }
    
    private fun animateLayoutChange(params: WindowManager.LayoutParams) {
        overlayView?.animate()
            ?.alpha(0.3f)
            ?.setDuration(200)
            ?.withEndAction {
                try {
                    windowManager.updateViewLayout(overlayView, params)
                    overlayView?.animate()
                        ?.alpha(1f)
                        ?.setDuration(300)
                        ?.start()
                } catch (e: Exception) {
                    Log.e(TAG, "Error in animation: ${e.message}")
                }
            }
            ?.start()
    }
    
    private fun snapToEdgeIfNeeded(params: WindowManager.LayoutParams) {
        val displayMetrics = resources.displayMetrics
        val screenWidth = displayMetrics.widthPixels
        val threshold = 50
        
        if (params.x < threshold) {
            params.x = 0
            try {
                windowManager.updateViewLayout(overlayView, params)
            } catch (e: Exception) {
                Log.e(TAG, "Error snapping to edge: ${e.message}")
            }
        } else if (params.x > screenWidth - threshold) {
            params.gravity = Gravity.TOP or Gravity.END
            params.x = 0
            try {
                windowManager.updateViewLayout(overlayView, params)
            } catch (e: Exception) {
                Log.e(TAG, "Error snapping to edge: ${e.message}")
            }
        }
    }
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        return START_STICKY
    }
    
    override fun onDestroy() {
        super.onDestroy()
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            audioManager.unregisterAudioPlaybackCallback(audioPlaybackCallback)
        }
        
        idleAnimationJob?.cancel()
        monitorJob?.cancel()
        audioMonitorJob?.cancel()
        scope.cancel()
        saveCurrentState()
        
        gifDrawable = null
        
        try {
            if (isViewAttached) {
                overlayView?.let {
                    windowManager.removeView(it)
                }
                isViewAttached = false
            }
        } catch (e: Exception) {
            Log.e(TAG, "Error in onDestroy: ${e.message}")
        }
    }
    
    override fun onBind(intent: Intent?): IBinder? = null
}

# 2. Create README
@"
# ChatGPT Avatar Overlay

An Android overlay app that displays an animated avatar during ChatGPT voice conversations.

## Features
- 🎭 Automatically appears when ChatGPT voice mode is active
- 👆 Tap to start/stop lip-sync animation
- 🎯 Drag to reposition anywhere on screen
- 🔄 Double-tap to cycle through display modes (corner, sidebar, ambient)
- 👻 Transparent background - avatar floats over ChatGPT

## How It Works
The app monitors your device's audio output. When it detects that voice mode is active (audio playing), the avatar overlay appears. You control the animation manually by tapping - this way it only animates when the LLM is actually speaking to you, not when you're speaking.

## Installation

### Prerequisites
- Android 8.0 (API 26) or higher
- Android Studio (for building from source)

### Build & Install
1. Clone this repository
2. Open in Android Studio
3. Build the APK:
```
   gradlew assembleDebug
```
4. Install on your device:
   - Find the APK at `app/build/outputs/apk/debug/app-debug.apk`
   - Transfer to your phone and install
5. Grant overlay permission when prompted
6. Enable the overlay from the app

## Usage
1. Open the ChatGPT app
2. Start a voice conversation
3. Avatar appears automatically as a placeholder
4. **Tap once** to start the lip-sync animation (when LLM is speaking)
5. **Tap again** to stop the animation (when you're speaking)
6. **Drag** to reposition the avatar
7. **Double-tap** to cycle through display modes:
   - Top corner (320x320)
   - Sidebar (120x300)
   - Ambient (fullscreen, very transparent)
8. Exit voice mode - avatar disappears

## Customization

### Using Your Own Avatar
Replace `app/src/main/res/raw/avatar_lipsync.gif` with your own animated GIF. The GIF should:
- Show lip movement/talking animation
- Be around 2-3 seconds long
- Work well when looped
- Ideally have a transparent or simple background

### Adjusting Size & Position
The app remembers your custom position. Default sizes:
- Top corner: 320x320 pixels
- Sidebar: 120x300 pixels  
- Ambient: Full screen

Edit these in `OverlayService.kt` in the `applyPositionMode()` function.

## Technical Details

### Architecture
- Service-based overlay using `TYPE_APPLICATION_OVERLAY`
- Audio monitoring via `AudioManager` and `AudioPlaybackCallback`
- GIF rendering with Glide library
- Foreground service (required for Android 14+)

### Permissions Required
- `SYSTEM_ALERT_WINDOW` - Display overlay
- `FOREGROUND_SERVICE` - Keep service running
- `FOREGROUND_SERVICE_MEDIA_PLAYBACK` - Audio monitoring
- `POST_NOTIFICATIONS` - Foreground service notification

## Troubleshooting

**Avatar doesn't appear:**
- Check overlay permission is granted
- Ensure voice mode is actually active in ChatGPT
- Try restarting the overlay service

**Animation won't stop:**
- Tap the avatar directly (not surrounding area)
- If stuck, exit and re-enter voice mode

**App crashes on start:**
- Ensure you're on Android 8.0+
- Check all permissions are granted
- Reinstall the app

## Known Limitations
- Cannot automatically distinguish between your voice and the LLM's voice
- Requires manual tap control for animation
- Only works when audio is actively playing through the device

## Future Enhancements
- [ ] Multiple avatar options
- [ ] Auto-detection of LLM speech (via accessibility service)
- [ ] Customizable animations
- [ ] Settings screen
- [ ] Support for other AI chat apps

## Credits
Built during a collaborative coding session. Avatar animation concept for enhancing AI voice interactions.

## License
MIT License - feel free to modify and use as you wish.
