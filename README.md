# Expo Camera and Notifications Guide

This guide will walk you through implementing camera functionality and push notifications in your Expo app. We'll cover everything from setup to implementation.

## Prerequisites

Before starting, make sure you have:
- Node.js installed (recommended version: 18.x or 20.x)
- Expo CLI installed (`npm install -g expo-cli`)
- A basic understanding of React Native and TypeScript

## Required Dependencies

Install these packages in your Expo project:

```bash
npx expo install expo-camera
npx expo install expo-notifications
npx expo install expo-media-library
npx expo install expo-device
npx expo install expo-constants
```

## Project Configuration

### Update app.json

Add the following configuration to your `app.json`:

```json
{
  "expo": {
    "plugins": [
      "expo-router",
      [
        "expo-camera",
        {
          "cameraPermission": "Allow $(PRODUCT_NAME) to access your camera.",
          "microphonePermission": "Allow $(PRODUCT_NAME) to access your microphone."
        }
      ],
      [
        "expo-media-library",
        {
          "photosPermission": "Allow $(PRODUCT_NAME) to access your photos.",
          "savePhotosPermission": "Allow $(PRODUCT_NAME) to save photos.",
          "isAccessMediaLocationEnabled": true
        }
      ],
      [
        "expo-notifications"
      ]
    ],
    "ios": {
      "infoPlist": {
        "NSCameraUsageDescription": "This app uses the camera to take photos.",
        "NSPhotoLibraryUsageDescription": "This app saves photos to your photo library.",
        "NSPhotoLibraryAddUsageDescription": "This app saves photos to your photo library."
      }
    },
    "android": {
      "permissions": [
        "CAMERA",
        "WRITE_EXTERNAL_STORAGE",
        "READ_EXTERNAL_STORAGE",
        "NOTIFICATIONS"
      ]
    }
  }
}
```

## Implementing Push Notifications

### 1. Create a Notifications Service

Create a new file `services/notifications.ts`:

```typescript
import * as Notifications from 'expo-notifications';
import { Platform } from 'react-native';

// Configure how notifications should be handled
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: false,
  }),
});

// Function to send an immediate notification
export async function schedulePushNotification(title: string, body: string) {
  await Notifications.scheduleNotificationAsync({
    content: {
      title,
      body,
    },
    trigger: null, // null means show immediately
  });
}

// Function to request notification permissions and get push token
export async function registerForPushNotificationsAsync() {
  let token;

  // Set up Android channel
  if (Platform.OS === 'android') {
    await Notifications.setNotificationChannelAsync('default', {
      name: 'default',
      importance: Notifications.AndroidImportance.MAX,
      vibrationPattern: [0, 250, 250, 250],
      lightColor: '#FF231F7C',
    });
  }

  // Request permission
  const { status: existingStatus } = await Notifications.getPermissionsAsync();
  let finalStatus = existingStatus;

  if (existingStatus !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }

  if (finalStatus !== 'granted') {
    alert('Failed to get push token for push notification!');
    return;
  }

  token = (await Notifications.getExpoPushTokenAsync({
    projectId: "your-project-id" // Replace with your project ID from app.json
  })).data;

  return token;
}
```

### 2. Using Notifications

Example of sending a test notification:

```typescript
const sendTestNotification = async () => {
  try {
    await schedulePushNotification(
      'Test Notification! 🔔',
      'This is a test notification from your app!'
    );
  } catch (error) {
    console.error('Error sending notification:', error);
  }
};
```

## Implementing Camera

### 1. Create a Camera Screen Component

```typescript
import { CameraView, CameraType, useCameraPermissions } from 'expo-camera';
import * as MediaLibrary from 'expo-media-library';
import { useState, useRef } from 'react';

export default function CameraScreen() {
  // Permissions hooks
  const [permission, requestPermission] = useCameraPermissions();
  const [mediaLibraryPermission, requestMediaLibraryPermission] = MediaLibrary.usePermissions();
  
  // State management
  const [facing, setFacing] = useState<CameraType>('back');
  const [lastPhotoUri, setLastPhotoUri] = useState<string | null>(null);
  const [isCapturing, setIsCapturing] = useState(false);
  
  // Camera reference
  const cameraRef = useRef<CameraView>(null);

  // Handle permissions
  if (!permission || !mediaLibraryPermission) {
    return <Text>Requesting permissions...</Text>;
  }

  if (!permission.granted || !mediaLibraryPermission.granted) {
    return (
      <View>
        <Text>We need camera and media library permissions</Text>
        <Button onPress={requestPermission} title="Grant camera permission" />
        <Button onPress={requestMediaLibraryPermission} title="Grant media library permission" />
      </View>
    );
  }

  // Function to take a picture
  const takePicture = async () => {
    if (!cameraRef.current || isCapturing) return;

    try {
      setIsCapturing(true);
      const photo = await cameraRef.current.takePictureAsync({
        quality: 1,
        exif: true
      });
      
      if (photo?.uri) {
        setLastPhotoUri(photo.uri);
        await MediaLibrary.saveToLibraryAsync(photo.uri);
        // Optional: Send notification when photo is taken
        await schedulePushNotification(
          'Photo Captured! 📸',
          'Your photo has been saved to the gallery'
        );
      }
    } catch (error) {
      console.error('Error taking picture:', error);
    } finally {
      setIsCapturing(false);
    }
  };

  return (
    <View style={{ flex: 1 }}>
      <CameraView 
        style={{ flex: 1 }} 
        facing={facing}
        ref={cameraRef}
      >
        {/* Camera UI */}
      </CameraView>
    </View>
  );
}
```

## Key Concepts

1. **Permissions**: Both camera and notifications require explicit user permissions. Always handle cases where permissions aren't granted.

2. **Camera Types**: The camera can be either front (`'front'`) or back (`'back'`). Use the `facing` prop to control this.

3. **Notification Triggers**: 
   - `null` trigger means show immediately
   - You can also schedule notifications for later using timestamps

4. **Platform Differences**:
   - iOS requires additional permission configuration in `app.json`
   - Android requires setting up notification channels

## Best Practices

1. **Error Handling**: Always wrap async operations in try-catch blocks.

2. **Permission States**: Handle all permission states (loading, denied, granted).

3. **Memory Management**: Clean up resources when components unmount.

4. **User Feedback**: Provide clear feedback when actions succeed or fail.

## Common Issues and Solutions

1. **Camera Not Working**:
   - Ensure permissions are granted
   - Check if another app is using the camera
   - Verify device has a camera

2. **Notifications Not Showing**:
   - Check if app has notification permissions
   - Verify device is not in silent mode
   - On Android, ensure notification channel is set up

3. **Permission Issues**:
   - Always request permissions at the appropriate time
   - Provide clear explanations for why permissions are needed
   - Handle cases where users deny permissions

## Testing

1. **Camera**:
   - Test on both physical devices and simulators
   - Test both front and back cameras
   - Verify photos are saved correctly

2. **Notifications**:
   - Test on both foreground and background states
   - Verify notification interactions work
   - Test on both iOS and Android devices

Remember to always test your implementation on both iOS and Android devices, as behavior can differ between platforms.