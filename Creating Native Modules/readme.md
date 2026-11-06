# Creating Native Modules for both Android and IOS.

## Steps to Follow
---

### Step 1: Create Native Module for Android (Java)

1.1 Create a Custom file name (**`CustomDocumentPickerModule.java`**) in your Android source directory:
`android/app/src/main/java/com/cc2022/CustomDocumentPickerModule.java`

```java
package com.cc2022;

import android.app.Activity;
import android.content.Intent;
import android.database.Cursor;
import android.net.Uri;
import android.provider.OpenableColumns;
import androidx.annotation.Nullable;

import com.facebook.react.bridge.ActivityEventListener;
import com.facebook.react.bridge.Arguments;
import com.facebook.react.bridge.Promise;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.bridge.ReactContextBaseJavaModule;
import com.facebook.react.bridge.ReactMethod;
import com.facebook.react.bridge.WritableMap;

import java.io.File;
import java.io.FileOutputStream;
import java.io.InputStream;

public class CustomDocumentPickerModule extends ReactContextBaseJavaModule implements ActivityEventListener {

    private static final int PICK_DOCUMENT_REQUEST_CODE = 1337;
    private Promise mPickerPromise;

    public CustomDocumentPickerModule(ReactApplicationContext reactContext) {
        super(reactContext);
        reactContext.addActivityEventListener(this);
    }

    @Override
    public String getName() {
        return "CustomDocumentPicker";
    }

    @ReactMethod
    public void pickDocument(Promise promise) {
        Activity currentActivity = getCurrentActivity();
        if (currentActivity == null) {
            promise.reject("E_ACTIVITY_DOES_NOT_EXIST", "Activity doesn't exist");
            return;
        }

        mPickerPromise = promise;

        try {
            Intent intent = new Intent(Intent.ACTION_OPEN_DOCUMENT);
            intent.addCategory(Intent.CATEGORY_OPENABLE);
            intent.setType("*/*");
            // Specify common document MIME types if needed, or leave */* for universal support
            currentActivity.startActivityForResult(intent, PICK_DOCUMENT_REQUEST_CODE);
        } catch (Exception e) {
            if (mPickerPromise != null) {
                mPickerPromise.reject("E_FAILED_TO_SHOW_PICKER", e.getMessage());
                mPickerPromise = null;
            }
        }
    }

    @Override
    public void onActivityResult(Activity activity, int requestCode, int resultCode, @Nullable Intent data) {
        if (requestCode != PICK_DOCUMENT_REQUEST_CODE) {
            return;
        }

        if (mPickerPromise == null) {
            return;
        }

        if (resultCode == Activity.RESULT_CANCELED) {
            mPickerPromise.reject("USER_CANCELLED", "User cancelled document selection");
            mPickerPromise = null;
            return;
        }

        if (resultCode == Activity.RESULT_OK && data != null) {
            Uri uri = data.getData();
            if (uri == null) {
                mPickerPromise.reject("E_INVALID_URI", "Selected file URI is null");
                mPickerPromise = null;
                return;
            }

            try {
                ReactApplicationContext context = getReactApplicationContext();
                String fileName = getFileName(uri);
                File cachedFile = copyFileToCache(context, uri, fileName);

                WritableMap map = Arguments.createMap();
                map.putString("uri", Uri.fromFile(cachedFile).toString());
                map.putString("name", fileName);
                map.putDouble("size", cachedFile.length());
                map.putString("type", context.getContentResolver().getType(uri));

                mPickerPromise.resolve(map);
            } catch (Exception e) {
                mPickerPromise.reject("E_PROCESSING_FILE", e.getMessage());
            } finally {
                mPickerPromise = null;
            }
        } else {
            mPickerPromise.reject("E_UNKNOWN_ERROR", "Unknown error while picking document");
            mPickerPromise = null;
        }
    }

    @Override
    public void onNewIntent(Intent intent) {
        // Not required for this implementation
    }

    private String getFileName(Uri uri) {
        String result = "document";
        Cursor cursor = getReactApplicationContext().getContentResolver().query(uri, null, null, null, null);
        try {
            if (cursor != null && cursor.moveToFirst()) {
                int nameIndex = cursor.getColumnIndex(OpenableColumns.DISPLAY_NAME);
                if (nameIndex != -1) {
                    result = cursor.getString(nameIndex);
                }
            }
        } finally {
            if (cursor != null) {
                cursor.close();
            }
        }
        return result;
    }

    private File copyFileToCache(ReactApplicationContext context, Uri uri, String fileName) throws Exception {
        InputStream inputStream = context.getContentResolver().openInputStream(uri);
        File cacheDir = context.getCacheDir();
        File outputFile = new File(cacheDir, System.currentTimeMillis() + "_" + fileName);
        FileOutputStream outputStream = new FileOutputStream(outputFile);

        byte[] buffer = new byte[4096]; // Optimized buffer size for 2GB RAM devices to prevent OOM
        int length;
        while ((length = inputStream.read(buffer)) > 0) {
            outputStream.write(buffer, 0, length);
        }

        outputStream.flush();
        outputStream.close();
        inputStream.close();
        return outputFile;
    }
}

```

1.2 Create the Custom Package registration file (**`CustomDocumentPickerPackage.java`**) in the same folder:
`android/app/src/main/java/com/cc2022/CustomDocumentPickerPackage.java`

```java
package com.cc2022;

import com.facebook.react.ReactPackage;
import com.facebook.react.bridge.NativeModule;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.uimanager.ViewManager;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class CustomDocumentPickerPackage implements ReactPackage {
    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }

    @Override
    public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
        List<NativeModule> modules = new ArrayList<>();
        modules.add(new CustomDocumentPickerModule(reactContext));
        return modules;
    }
}

```

---

### Step 2: Create Native Module for iOS (Objective-C)

2.1 Create custom file name ( **`CustomDocumentPicker.h`** ) in `ios/cc2022/`:

```objc
#import <React/RCTBridgeModule>
#import <UIKit/UIKit.h>

@interface CustomDocumentPicker : NSObject <RCTBridgeModule, UIDocumentPickerDelegate>
@end

```

2.2 Create custom file name (**`CustomDocumentPicker.m`**) in `ios/cc2022/`:

```objc
#import "CustomDocumentPicker.h"

@implementation CustomDocumentPicker {
    RCTPromiseResolveBlock _resolve;
    RCTPromiseRejectBlock _reject;
}

RCT_EXPORT_MODULE();

RCT_EXPORT_METHOD(pickDocument:(RCTPromiseResolveBlock)resolve
                  rejecter:(RCTPromiseRejectBlock)reject)
{
    if (_resolve) {
        reject(@"E_PICKER_BUSY", @"Another document picker is already active", nil);
        return;
    }

    _resolve = resolve;
    _reject = reject;

    dispatch_async(dispatch_get_main_queue(), ^{
        UIDocumentPickerViewController *documentPicker = [[UIDocumentPickerViewController alloc] initWithDocumentTypes:@[@"public.item"] inMode:UIDocumentPickerModeImport];
        documentPicker.delegate = self;
        documentPicker.modalPresentationStyle = UIModalPresentationFullScreen;

        UIViewController *rootViewController = [UIApplication sharedApplication].delegate.window.rootViewController;
        [rootViewController presentViewController:documentPicker animated:YES completion:nil];
    });
}

#pragma mark - UIDocumentPickerDelegate

- (void)documentPicker:(UIDocumentPickerViewController *)controller didPickDocumentsAtURLs:(NSArray <NSURL *> *)urls {
    if (!_resolve) return;

    NSURL *sourceURL = [urls firstObject];
    if (!sourceURL) {
        _reject(@"E_INVALID_URL", @"No file URL returned", nil);
        _cleanUp();
        return;
    }

    NSError *error = nil;
    NSString *fileName = [sourceURL lastPathComponent];
    NSString *tempDir = NSTemporaryDirectory();
    NSString *destPath = [tempDir stringByAppendingPathComponent:[NSString stringWithFormat:@"%f_%@", [[NSDate date] timeIntervalSince1970], fileName]];
    NSURL *destURL = [NSURL fileURLWithPath:destPath];

    if ([[NSFileManager defaultManager] fileExistsAtPath:destPath]) {
        [[NSFileManager defaultManager] removeItemAtPath:destPath error:nil];
    }

    BOOL success = [[NSFileManager defaultManager] copyItemAtURL:sourceURL toURL:destURL error:&error];
    if (!success || error) {
        _reject(@"E_COPY_FAILED", error.localizedDescription ?: @"Failed to copy picked file", error);
        _cleanUp();
        return;
    }

    NSDictionary *fileAttributes = [[NSFileManager defaultManager] attributesOfItemAtPath:destPath error:nil];
    NSNumber *fileSize = [fileAttributes objectForKey:NSFileSize] ?: @0;

    NSDictionary *result = @{
        @"uri": [destURL absoluteString],
        @"name": fileName ?: @"document",
        @"size": fileSize,
        @"type": [self mimeTypeForPath:destPath]
    };

    _resolve(result);
    _cleanUp();
}

- (void)documentPickerWasCancelled:(UIDocumentPickerViewController *)controller {
    if (_reject) {
        _reject(@"USER_CANCELLED", @"User cancelled document selection", nil);
    }
    _cleanUp();
}

- (void)_cleanUp {
    _resolve = nil;
    _reject = nil;
}

- (NSString *)mimeTypeForPath:(NSString *)path {
    // Basic fallback MIME type resolution
    CFStringRef extension = (__bridge CFStringRef)[path pathExtension];
    CFStringRef UTI = UTTypeCreatePreferredIdentifierForTag(kUTTagClassFilenameExtension, extension, NULL);
    if (!UTI) return @"application/octet-stream";
    CFStringRef mimeType = UTTypeCopyPreferredTagWithClass(UTI, kUTTagClassMIMEType);
    CFRelease(UTI);
    if (!mimeType) return @"application/octet-stream";
    return (__bridge_transfer NSString *)mimeType;
}

@end

```

---

### Step 3: Use that Native Module in your Project Components (`MyHousePointsDocumentScreen.js`)

Here is your bulletproof, optimized component using the native module directly without any third-party document pickers:

```jsx
import AsyncStorage from "@react-native-async-storage/async-storage";
import { useFocusEffect } from "@react-navigation/native";
import { useCallback, useEffect, useState } from "react";
import {
  Linking,
  Modal,
  SafeAreaView,
  ScrollView,
  Text,
  TouchableOpacity,
  View,
  NativeModules,
  Platform
} from "react-native";
import Toast from "react-native-simple-toast";
import FontAwesome5 from "react-native-vector-icons/FontAwesome5";
import { useDispatch, useSelector } from "react-redux";
import AppStatusBar from "../components/AppStatusBar";
import { Loader } from "../components/Loader";
import { SectionHeader } from "../components/SectionHeader";
import StaticBottomTabs from "../components/StaticBottomTabs";
import { COLORS } from "../constants";
import {
  deleteHousePointDocument,
  fetchHousePointDocuments,
} from "../redux/actions/MyHousePointsListState";
import appStyle, { default as styles } from "../styles/AppStyles";
import { postRequest } from "../utils/Handlers";

const { CustomDocumentPicker } = NativeModules;

const MyHousePointsDocumentScreen = ({ navigation, route }) => {
  const dispatch = useDispatch();
  const { loggedMember } = useSelector((state) => state.AuthState);

  const refID = route?.params?.idx || 0;
  const [loader, setLoader] = useState(true);
  const [documentData, setDocumentData] = useState(null);
  const [docCount, setDocCount] = useState(0);
  const [userimageurl, setUserimageurl] = useState(null);

  // Custom Delete Modal State
  const [deleteModalVisible, setDeleteModalVisible] = useState(false);

  useEffect(() => {
    AsyncStorage.getItem("loginWithCode").then((val) => {
      if (val) {
        try {
          setUserimageurl(JSON.parse(val));
        } catch (e) {
          setUserimageurl(null);
        }
      }
    });
  }, []);

  const loadDocuments = useCallback(() => {
    setLoader(true);
    const iACID = loggedMember?.MobilDATA?.AcademicYearID || 0;
    const userId = loggedMember?.USERRECORDID || 0;

    fetchHousePointDocuments(iACID, userId, refID)
      .then((resp) => {
        if (resp && Array.isArray(resp) && resp.length > 0) {
          const item = resp[0];
          setDocumentData(item);
          setDocCount(item?.DocX && item.DocX.trim().length > 0 ? 1 : 0);
        } else {
          setDocumentData(null);
          setDocCount(0);
        }
        setLoader(false);
      })
      .catch((err) => {
        setLoader(false);
        Toast.showWithGravity(
          err.message || "Failed to load documents",
          Toast.LONG,
          Toast.BOTTOM,
        );
      });
  }, [loggedMember, refID]);

  useFocusEffect(
    useCallback(() => {
      loadDocuments();
    }, [loadDocuments]),
  );

  // Native Document Picker Integration (Zero 3rd party dependencies)
  const handleAddDocument = async () => {
    if (docCount > 0) {
      Toast.showWithGravity(
        "A document already exists. Can not add more than one document.",
        Toast.LONG,
        Toast.BOTTOM,
      );
      return;
    }

    try {
      setLoader(true);
      const pickerResult = await CustomDocumentPicker.pickDocument();
      if (!pickerResult || !pickerResult.uri) {
        setLoader(false);
        return;
      }

      const fileUri = pickerResult.uri;
      const fileName = pickerResult.name || "document.pdf";
      const iUserId = loggedMember?.USERRECORDID || 0;
      const iACID = loggedMember?.MobilDATA?.AcademicYearID || 0;

      const formattedUri =
        Platform.OS === "ios" && !fileUri.startsWith("file://")
          ? `file://${fileUri}`
          : fileUri;

      const xhr = new XMLHttpRequest();
      xhr.onload = function () {
        const reader = new FileReader();
        reader.onloadend = async () => {
          const base64Result = reader.result;
          const base64Data = base64Result.includes(",")
            ? base64Result.split(",")[1]
            : base64Result;

          const imageDetails = {
            ImageString: base64Data,
            AcademicYearID: iACID,
            USERRECORDID: iUserId,
            FileName: fileName,
            ReferenceID: refID,
          };

          try {
            const uploadParams = {
              url: "Mobile2/APISaveStudentPointsDocument",
              data: imageDetails,
            };
            await postRequest(uploadParams);
            setLoader(false);
            Toast.showWithGravity(
              "The document has been uploaded successfully.",
              Toast.LONG,
              Toast.BOTTOM,
            );
            loadDocuments();
          } catch (uploadErr) {
            setLoader(false);
            Toast.showWithGravity(
              "Unable to upload the document. Sorry for the inconvenience.",
              Toast.LONG,
              Toast.BOTTOM,
            );
          }
        };
        reader.readAsDataURL(xhr.response);
      };
      xhr.onerror = function () {
        setLoader(false);
        Toast.showWithGravity(
          "Failed to process the selected file.",
          Toast.LONG,
          Toast.BOTTOM,
        );
      };
      xhr.open("GET", formattedUri);
      xhr.responseType = "blob";
      xhr.send();
    } catch (err) {
      setLoader(false);
      if (err?.code !== "USER_CANCELLED") {
        Toast.showWithGravity(
          err?.message || "File selection failed",
          Toast.LONG,
          Toast.BOTTOM,
        );
      }
    }
  };

  const confirmDeleteDocument = () => {
    setDeleteModalVisible(false);
    setLoader(true);
    const iUserId = loggedMember?.USERRECORDID || 0;

    deleteHousePointDocument(refID, iUserId)
      .then((resp) => {
        setLoader(false);
        if (resp === "Success" || resp === true) {
          Toast.showWithGravity(
            "The document has been deleted successfully.",
            Toast.LONG,
            Toast.BOTTOM,
          );
          loadDocuments();
        } else {
          Toast.showWithGravity(
            "Error occurred while deleting the document.",
            Toast.LONG,
            Toast.BOTTOM,
          );
        }
      })
      .catch((err) => {
        setLoader(false);
        Toast.showWithGravity("Delete failed", Toast.LONG, Toast.BOTTOM);
      });
  };

  const handleOpenDocument = (fileName) => {
    if (!fileName || !userimageurl?.apiUrl) {
      Toast.showWithGravity(
        "Unable to resolve document path",
        Toast.LONG,
        Toast.BOTTOM,
      );
      return;
    }
    const rootPath = userimageurl.apiUrl.replace("api/", "");
    const strPath = `${rootPath}TempFile/house/${fileName}`;
    Linking.openURL(strPath).catch(() => {
      Toast.showWithGravity(
        "Unable to open document",
        Toast.LONG,
        Toast.BOTTOM,
      );
    });
  };

  const statusX = documentData?.StatusX || 0;
  const hasDoc = documentData?.DocX && documentData.DocX.trim().length > 0;

  let statusBadgeText = "Draft";
  let statusBadgeStyle = styles.pagelabel12Draft;

  if (statusX === 1) {
    statusBadgeText = "Approved";
    statusBadgeStyle = styles.pagelabel12Complete;
  } else if (statusX === 4) {
    statusBadgeText = "Rejected";
    statusBadgeStyle = styles.pagelabel12Rejected;
  } else if (statusX === 3) {
    statusBadgeText = "Pending Approval";
    statusBadgeStyle = styles.pagelabel12Pending;
  }

  const showDeleteBtn =
    (statusX === 2 || statusX === 4 || statusX === 0) && statusX !== 3;

  return (
    <>
      <SafeAreaView style={{ flex: 1, backgroundColor: COLORS.background }}>
        <View style={appStyle.MainContainer}>
          <AppStatusBar colorPalete="WHITE" bg={COLORS.white} />
          {loader && <Loader />}

          {/* Header */}
          <SectionHeader
            title="MY HOUSE: REQUESTS: DOCUMENTS"
            onBackPress={() => navigation.navigate("HousePointsList")}
          />

          <ScrollView showsVerticalScrollIndicator={false}>
            {/* Document Card Panel */}
            <View style={styles.hpanel}>
              <View style={[styles.panelBody, { padding: 15 }]}>
                {hasDoc ? (
                  <>
                    <Text
                      style={[
                        styles.f14,
                        { fontWeight: "bold", color: "#000", marginBottom: 6 },
                      ]}
                      numberOfLines={2}
                    >
                      {documentData.Descx}
                    </Text>
                    <Text
                      style={[styles.f12, { marginBottom: 12, color: "#555" }]}
                    >
                      Status:{" "}
                      <Text style={statusBadgeStyle}>{statusBadgeText}</Text>
                    </Text>

                    <View
                      style={[
                        styles.fdRow,
                        {
                          flexWrap: "wrap",
                          gap: 10,
                          marginTop: 4,
                          alignItems: "center",
                        },
                      ]}
                    >
                      <TouchableOpacity
                        style={[
                          styles.crbgInfo,
                          {
                            paddingHorizontal: 16,
                            height: 36,
                            backgroundColor: COLORS.darkskyblue || "#0288d1",
                            justifyContent: "center",
                            alignItems: "center",
                            borderRadius: 4,
                            minWidth: 90,
                          },
                        ]}
                        onPress={() => handleOpenDocument(documentData.DocX)}
                      >
                        <Text
                          style={[
                            styles.buttontext10,
                            { fontSize: 12, color: "#fff" },
                          ]}
                        >
                          <FontAwesome5 name="info-circle" size={11} /> View
                        </Text>
                      </TouchableOpacity>

                      {showDeleteBtn && (
                        <TouchableOpacity
                          style={[
                            styles.crbgDelete,
                            {
                              paddingHorizontal: 16,
                              height: 36,
                              backgroundColor: "#e57358",
                              justifyContent: "center",
                              alignItems: "center",
                              borderRadius: 4,
                              minWidth: 90,
                            },
                          ]}
                          onPress={() => setDeleteModalVisible(true)}
                        >
                          <Text
                            style={[
                              styles.buttontext10,
                              { fontSize: 12, color: "#fff" },
                            ]}
                          >
                            <FontAwesome5 name="trash-alt" size={11} /> Delete
                          </Text>
                        </TouchableOpacity>
                      )}
                    </View>
                  </>
                ) : (
                  <View style={{ padding: 15, alignItems: "center" }}>
                    <Text style={styles.emptyText}>No documents found.</Text>
                  </View>
                )}
              </View>
            </View>

            <View
              style={{
                marginVertical: 12,
                height: 1,
                backgroundColor: "#617dd1",
              }}
            />

            {/* Bottom Action Row */}
            {docCount === 0 && (
              <View
                style={[
                  styles.fdRow,
                  {
                    marginVertical: 10,
                    paddingHorizontal: 4,
                    flexWrap: "wrap",
                    gap: 10,
                    justifyContent: "flex-end",
                  },
                ]}
              >
                <TouchableOpacity
                  style={[
                    styles.submitButton,
                    {
                      height: 40,
                      paddingHorizontal: 16,
                      justifyContent: "center",
                      alignItems: "center",
                      backgroundColor: COLORS.primary,
                      borderRadius: 4,
                      minWidth: 120,
                    },
                  ]}
                  onPress={handleAddDocument}
                >
                  <Text
                    style={[styles.submitText, { fontSize: 13, color: "#fff" }]}
                  >
                    <FontAwesome5 name="plus" size={11} /> Add Documents
                  </Text>
                </TouchableOpacity>
              </View>
            )}

            <View style={{ height: 60 }}></View>
          </ScrollView>
        </View>

        {/* CUSTOM DELETE CONFIRMATION MODAL */}
        <Modal
          visible={deleteModalVisible}
          transparent={true}
          animationType="fade"
          onRequestClose={() => setDeleteModalVisible(false)}
        >
          <View style={styles.ModalContainer}>
            <View
              style={[
                styles.ModalContent,
                {
                  alignItems: "center",
                  paddingVertical: 30,
                  paddingHorizontal: 20,
                  width: "85%",
                  maxWidth: 340,
                  alignSelf: "center",
                },
              ]}
            >
              <View
                style={{
                  width: 80,
                  height: 80,
                  borderRadius: 40,
                  borderWidth: 3,
                  borderColor: "#f6ad55",
                  justifyContent: "center",
                  alignItems: "center",
                  marginBottom: 20,
                }}
              >
                <FontAwesome5 name="exclamation" size={40} color="#f6ad55" />
              </View>

              <Text
                style={[
                  styles.f22,
                  {
                    fontWeight: "bold",
                    color: "#333",
                    marginBottom: 12,
                    textAlign: "center",
                  },
                ]}
              >
                Are you sure?
              </Text>
              <Text
                style={[
                  styles.f14,
                  { textAlign: "center", color: "#666", marginBottom: 4 },
                ]}
              >
                Do you want delete the document?
              </Text>
              <Text
                style={[
                  styles.f14,
                  { textAlign: "center", color: "#666", marginBottom: 25 },
                ]}
              >
                The changes can not be reverted.
              </Text>

              <View
                style={[
                  styles.fdRow,
                  { justifyContent: "center", width: "100%", gap: 10 },
                ]}
              >
                <TouchableOpacity
                  style={{
                    backgroundColor: "#cbd5e1",
                    paddingVertical: 12,
                    paddingHorizontal: 25,
                    borderRadius: 6,
                    marginRight: 10,
                    flex: 1,
                    alignItems: "center",
                  }}
                  onPress={() => setDeleteModalVisible(false)}
                >
                  <Text
                    style={[
                      styles.f16,
                      { color: "#334155", fontWeight: "bold" },
                    ]}
                  >
                    No
                  </Text>
                </TouchableOpacity>

                <TouchableOpacity
                  style={{
                    backgroundColor: "#e57358",
                    paddingVertical: 12,
                    paddingHorizontal: 25,
                    borderRadius: 6,
                    marginLeft: 10,
                    flex: 1,
                    alignItems: "center",
                  }}
                  onPress={confirmDeleteDocument}
                >
                  <Text
                    style={[
                      styles.f16,
                      { color: "#ffffff", fontWeight: "bold" },
                    ]}
                  >
                    Yes
                  </Text>
                </TouchableOpacity>
              </View>
            </View>
          </View>
        </Modal>
      </SafeAreaView>
      <StaticBottomTabs navigation={navigation} routeName={route.name} />
    </>
  );
};

export default MyHousePointsDocumentScreen;

```

---

### Step 4: Step-by-Step Implementation in your Project

Follow these steps precisely to integrate the native module and clean up your project:

#### Step 4.1: Remove the Deprecated Dependency

1. Open your terminal in the root project folder (`cc2022`).
2. Uninstall `react-native-document-picker`:
```bash
npm uninstall react-native-document-picker

```


*(or `yarn remove react-native-document-picker`)*

#### Step 4.2: Register the Android Native Module

1. Create `CustomDocumentPickerModule.java` and `CustomDocumentPickerPackage.java` inside `android/app/src/main/java/com/cc2022/` (as provided in Part 1).
2. Open your `MainApplication.java` (located at `android/app/src/main/java/com/cc2022/MainApplication.java`), and register the package inside `getPackages()`:
```java
import com.cc2022.CustomDocumentPickerPackage; // <-- Add import

// Inside getPackages() list:
packages.add(new CustomDocumentPickerPackage());

```



#### Step 4.3: Register the iOS Native Module

1. Create `CustomDocumentPicker.h` and `CustomDocumentPicker.m` inside `ios/cc2022/` (as provided in Part 2).
2. Open Xcode (`ios/cc2022.xcworkspace`), right-click the `cc2022` folder group, choose **Add Files to "cc2022"...**, and select your newly created `CustomDocumentPicker.h` and `CustomDocumentPicker.m` files (ensure "Create groups" and target `cc2022` are checked).

#### Step 4.4: Clean Build and Run

Then clear caches and perform a clean build:

* **Android:**
```bash
cd android && ./gradlew clean && cd ..
npx react-native run-android

```


* **iOS:**
```bash
cd ios && pod install && cd ..
npx react-native run-ios

```
