# Android Build Fix - Step by Step Commands

## Issue: Package Name Mismatch (com.myapp vs com.b2c.crm)

When you encounter build errors related to package name mismatches in React Native autolinking, follow these steps:

### Step 1: Navigate to Android Directory
```powershell
cd android
```

### Step 2: Clean the Build
```powershell
.\gradlew clean
```

### Step 3: Go Back to Project Root
```powershell
cd ..
```

### Step 4: Run Android Build
```powershell
npm run android
```

---

## Alternative: Complete Clean (If Step 2 doesn't work)

If the above doesn't work, you may need to manually clean the build directories:

### Step 1: Navigate to Android Directory
```powershell
cd android
```

### Step 2: Clean Build Directories (PowerShell)
```powershell
if (Test-Path app\build) { Remove-Item -Recurse -Force app\build -ErrorAction SilentlyContinue }
if (Test-Path build) { Remove-Item -Recurse -Force build -ErrorAction SilentlyContinue }
if (Test-Path .gradle) { Remove-Item -Recurse -Force .gradle -ErrorAction SilentlyContinue }
```

### Step 3: Clean with Gradle
```powershell
.\gradlew clean
```

### Step 4: Go Back to Project Root
```powershell
cd ..
```

### Step 5: Run Android Build
```powershell
npm run android
```

---

## Quick One-Liner (From Project Root)

```powershell
cd android; .\gradlew clean; cd ..; npm run android
```

---

## What This Fixes

- **Package name mismatch errors** in autolinking
- **Stale build artifacts** that reference old package names
- **BuildConfig errors** where `com.myapp.BuildConfig` is referenced instead of `com.b2c.crm.BuildConfig`

---

## Prevention Tips

1. Always run `gradlew clean` after changing package names or applicationId
2. If you rename your app package, make sure to update:
   - `android/app/build.gradle` (namespace and applicationId)
   - `android/app/src/main/java/com/...` (package structure)
   - `android/app/src/main/AndroidManifest.xml` (if needed)

---

## Notes

- The `clean` command removes all build artifacts and forces React Native to regenerate autolinking files
- This ensures the autolinking system uses the correct package name from `build.gradle`
- On Windows PowerShell, use semicolons (`;`) to chain commands, not `&&`

