# carrental


Below is the full implementation of the AAOS Speed Monitoring System using AIDL, covering both Server App (which runs as a service) and Client App (which interacts with the service).
Features:
 AIDL Service for inter-process communication
 Foreground Service in Server App
 Client App to set and retrieve speed limits
 Callback mechanism to notify speed violations
 Simulated speed monitoring
________________________________________
1️ Server App (SpeedMonitorService)
This app runs as a background service in the vehicle, monitoring the speed and sending notifications when limits are exceeded.
________________________________________
Step 1: Create the AIDL Interface
AidlInterface: ISpeedMonitorService.aidl
Create this file in src/main/aidl/com/example/carrentalservice/




    package com.example.carrentalservice;
        interface ISpeedMonitorService {
           void setSpeedLimit(String renterId, int speedLimit);
           int getSpeedLimit(String renterId);
           void registerListener(ISpeedMonitorCallback listener);
        }




AidlInterface: ISpeedMonitorCallback.aidl
This callback interface will notify the client when speed exceeds the limit.
    package com.example.carrentalservice;
    interface ISpeedMonitorCallback {
         void onSpeedExceeded(String renterId, int currentSpeed);
    }
________________________________________
Step 2: Implement the Service
Service Class: SpeedMonitorService.kt




    package com.example.carrentalservice

    import android.app.Service
    import android.content.Intent
    import android.os.IBinder
    import android.util.Log
    import kotlinx.coroutines.*

    class SpeedMonitorService : Service() {
    
        private val rentersSpeedLimit = mutableMapOf<String, Int>()
        private var listener: ISpeedMonitorCallback? = null
        private var isMonitoring = false
    
        private val serviceScope = CoroutineScope(Dispatchers.IO)
    
        private val binder = object : ISpeedMonitorService.Stub() {
            override fun setSpeedLimit(renterId: String, speedLimit: Int) {
                rentersSpeedLimit[renterId] = speedLimit
                Log.d("SpeedMonitorService", "Speed limit set: $speedLimit km/h for $renterId")
            }
    
            override fun getSpeedLimit(renterId: String): Int {
                return rentersSpeedLimit[renterId] ?: 0
            }
    
            override fun registerListener(callback: ISpeedMonitorCallback) {
                listener = callback
            }
        }
    
        override fun onBind(intent: Intent?): IBinder {
            startMonitoringSpeed()
            return binder
        }
    
        private fun startMonitoringSpeed() {
            if (isMonitoring) return
            isMonitoring = true
            serviceScope.launch {
                while (isMonitoring) {
                    val renterId = "testRenter" // Simulated renter ID
                    val currentSpeed = getCurrentSpeed()
                    val speedLimit = rentersSpeedLimit[renterId] ?: 100 // Default 100 km/h
    
                    if (currentSpeed > speedLimit) {
                        Log.w("SpeedMonitorService", "Speed exceeded! Current: $currentSpeed, Limit: $speedLimit")
                        listener?.onSpeedExceeded(renterId, currentSpeed)
                        sendFirebaseNotification(renterId, currentSpeed)
                    }
                    delay(5000) // Check every 5 seconds
                }
            }
        }
    
        private fun getCurrentSpeed(): Int {
            return (80..150).random() // Simulated speed data
        }
    
        private fun sendFirebaseNotification(renterId: String, speed: Int) {
            // Placeholder for Firebase notification
            Log.d("SpeedMonitorService", "Sending Firebase alert for $renterId exceeding speed: $speed")
        }
    
        override fun onDestroy() {
            super.onDestroy()
            isMonitoring = false
            serviceScope.cancel()
        }
    }
________________________________________


2️ Client App (SpeedMonitorClient)
The Client App binds to the SpeedMonitorService and interacts with it.
________________________________________
Step 1: Define the Client Activity
File: MainActivity.kt
        
        
        
        
        package com.example.speedmonitorclient
        
        import android.app.Activity
        import android.content.ComponentName
        import android.content.Context
        import android.content.Intent
        import android.content.ServiceConnection
        import android.os.Bundle
        import android.os.IBinder
        import android.util.Log
        import android.widget.Button
        import android.widget.EditText
        import android.widget.TextView
        import com.example.carrentalservice.ISpeedMonitorService
        import com.example.carrentalservice.ISpeedMonitorCallback
        
        class MainActivity : Activity() {
        
            private var speedMonitorService: ISpeedMonitorService? = null
            private var isBound = false
        
            private val connection = object : ServiceConnection {
                override fun onServiceConnected(className: ComponentName, service: IBinder) {
                    speedMonitorService = ISpeedMonitorService.Stub.asInterface(service)
                    isBound = true
                }
        
                override fun onServiceDisconnected(className: ComponentName) {
                    speedMonitorService = null
                    isBound = false
                }
            }
        
            private val callback = object : ISpeedMonitorCallback.Stub() {
                override fun onSpeedExceeded(renterId: String, currentSpeed: Int) {
                    runOnUiThread {
                        findViewById<TextView>(R.id.textViewStatus).text =
                            "ALERT! $renterId exceeded speed: $currentSpeed km/h"
                    }
                }
            }
        
            override fun onCreate(savedInstanceState: Bundle?) {
                super.onCreate(savedInstanceState)
                setContentView(R.layout.activity_main)
        
                val editRenterId = findViewById<EditText>(R.id.editRenterId)
                val editSpeedLimit = findViewById<EditText>(R.id.editSpeedLimit)
                val btnSetSpeed = findViewById<Button>(R.id.btnSetSpeed)
                val btnGetSpeed = findViewById<Button>(R.id.btnGetSpeed)
                val textViewStatus = findViewById<TextView>(R.id.textViewStatus)
        
                val intent = Intent()
                intent.setClassName("com.example.carrentalservice", "com.example.carrentalservice.SpeedMonitorService")
                bindService(intent, connection, Context.BIND_AUTO_CREATE)
        
                btnSetSpeed.setOnClickListener {
                    val renterId = editRenterId.text.toString()
                    val speedLimit = editSpeedLimit.text.toString().toInt()
                    if (isBound) {
                        speedMonitorService?.setSpeedLimit(renterId, speedLimit)
                        speedMonitorService?.registerListener(callback)
                    }
                }
        
                btnGetSpeed.setOnClickListener {
                    val renterId = editRenterId.text.toString()
                    if (isBound) {
                        val speedLimit = speedMonitorService?.getSpeedLimit(renterId)
                        textViewStatus.text = "Speed Limit for $renterId: ${speedLimit ?: "Not Set"} km/h"
                    }
                }
            }
        
            override fun onDestroy() {
                super.onDestroy()
                if (isBound) {
                    unbindService(connection)
                    isBound = false
                }
            }
        }

________________________________________
3️ AndroidManifest.xml (Server & Client)
Server App
        <service android:name=".SpeedMonitorService"
        android:exported="true"
        android:permission="android.permission.BIND_REMOTE_SERVICE">
        <intent-filter>
            <action android:name="com.example.carrentalservice.SPEED_MONITOR_SERVICE" />
        </intent-filter>
        </service>

Client App
        <uses-permission android:name="android.permission.BIND_REMOTE_SERVICE"/>

