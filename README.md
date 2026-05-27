# SecureStorageLabJava — TP Stockage Android
---
# ✅ TÂCHE 1 — Création du projet + dépendances
## Étapes
- Android Studio → New Project
- Empty Views Activity
- Nom : SecureStorageLabJava
- Langage : Java
- Min SDK : 24
## Gradle (app/build.gradle)
```gradle
dependencies {
    implementation "androidx.security:security-crypto:1.1.0-alpha06"
}

Vérification

* Sync Gradle OK
* Build OK
* Application lance écran vide

⸻

✅ TÂCHE 2 — SharedPreferences (non sécurisé)

AppPrefs.java

package com.example.securestoragejava.prefs;
import android.content.Context;
import android.content.SharedPreferences;
public final class AppPrefs {
    private static final String PREFS_NAME = "app_prefs";
    private static final String KEY_NAME = "pref_name";
    private static final String KEY_LANG = "pref_lang";
    private static final String KEY_THEME = "pref_theme";
    private AppPrefs() {}
    public static boolean save(Context context, String name, String lang, String theme, boolean sync) {
        SharedPreferences prefs = context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE);
        SharedPreferences.Editor editor = prefs.edit()
                .putString(KEY_NAME, name)
                .putString(KEY_LANG, lang)
                .putString(KEY_THEME, theme);
        if (sync) return editor.commit();
        editor.apply();
        return true;
    }
    public static Triple load(Context context) {
        SharedPreferences prefs = context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE);
        return new Triple(
                prefs.getString(KEY_NAME, ""),
                prefs.getString(KEY_LANG, "fr"),
                prefs.getString(KEY_THEME, "system")
        );
    }
    public static void clear(Context context) {
        prefs.edit().clear().apply();
    }
    public static final class Triple {
        public final String name;
        public final String lang;
        public final String theme;
        public Triple(String name, String lang, String theme) {
            this.name = name;
            this.lang = lang;
            this.theme = theme;
        }
    }
}

⸻

✅ TÂCHE 3 — Stockage sécurisé

SecurePrefs.java

package com.example.securestoragejava.prefs;
import android.content.Context;
import android.content.SharedPreferences;
import androidx.security.crypto.EncryptedSharedPreferences;
import androidx.security.crypto.MasterKey;
public final class SecurePrefs {
    private static final String PREFS_NAME = "secure_prefs";
    private static final String KEY_API_TOKEN = "secure_api_token";
    private SecurePrefs() {}
    private static SharedPreferences securePrefs(Context context) throws Exception {
        MasterKey masterKey = new MasterKey.Builder(context)
                .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
                .build();
        return EncryptedSharedPreferences.create(
                context,
                PREFS_NAME,
                masterKey,
                EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
                EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
        );
    }
    public static void saveToken(Context context, String token) throws Exception {
        securePrefs(context).edit().putString(KEY_API_TOKEN, token).apply();
    }
    public static String loadToken(Context context) throws Exception {
        return securePrefs(context).getString(KEY_API_TOKEN, "");
    }
    public static void clear(Context context) throws Exception {
        securePrefs(context).edit().clear().apply();
    }
}

⸻

✅ TÂCHE 4 — Internal Storage + JSON

InternalTextStore.java

package com.example.securestoragejava.files;
import android.content.Context;
import java.io.*;
import java.nio.charset.StandardCharsets;
public final class InternalTextStore {
    public static void writeUtf8(Context context, String fileName, String content) throws Exception {
        try (FileOutputStream fos = context.openFileOutput(fileName, Context.MODE_PRIVATE)) {
            fos.write(content.getBytes(StandardCharsets.UTF_8));
        }
    }
    public static String readUtf8(Context context, String fileName) throws Exception {
        try (FileInputStream fis = context.openFileInput(fileName)) {
            return new String(fis.readAllBytes(), StandardCharsets.UTF_8);
        }
    }
}

Student.java

package com.example.securestoragejava.model;
public class Student {
    public final int id;
    public final String name;
    public final int age;
    public Student(int id, String name, int age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }
}

StudentsJsonStore.java

package com.example.securestoragejava.files;
import android.content.Context;
import com.example.securestoragejava.model.Student;
import org.json.*;
import java.util.*;
public final class StudentsJsonStore {
    public static void save(Context context, List<Student> students) throws Exception {
        JSONArray arr = new JSONArray();
        for (Student s : students) {
            JSONObject obj = new JSONObject();
            obj.put("id", s.id);
            obj.put("name", s.name);
            obj.put("age", s.age);
            arr.put(obj);
        }
        try (var fos = context.openFileOutput("students.json", Context.MODE_PRIVATE)) {
            fos.write(arr.toString().getBytes());
        }
    }
    public static List<Student> load(Context context) {
        try (var fis = context.openFileInput("students.json")) {
            String json = new String(fis.readAllBytes());
            JSONArray arr = new JSONArray(json);
            List<Student> list = new ArrayList<>();
            for (int i = 0; i < arr.length(); i++) {
                JSONObject o = arr.getJSONObject(i);
                list.add(new Student(
                        o.getInt("id"),
                        o.getString("name"),
                        o.getInt("age")
                ));
            }
            return list;
        } catch (Exception e) {
            return List.of();
        }
    }
}

⸻

✅ TÂCHE 5 — Cache

CacheStore.java

package com.example.securestoragejava.cache;
import android.content.Context;
import java.io.File;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
public final class CacheStore {
    public static void write(Context context, String fileName, String content) throws Exception {
        File file = new File(context.getCacheDir(), fileName);
        Files.writeString(file.toPath(), content, StandardCharsets.UTF_8);
    }
    public static String read(Context context, String fileName) throws Exception {
        File file = new File(context.getCacheDir(), fileName);
        if (!file.exists()) return null;
        return Files.readString(file.toPath(), StandardCharsets.UTF_8);
    }
}

⸻

✅ TÂCHE 6 — External Storage

ExternalAppFilesStore.java

package com.example.securestoragejava.external;
import android.content.Context;
import java.io.File;
import java.nio.file.Files;
import java.nio.charset.StandardCharsets;
public final class ExternalAppFilesStore {
    public static String write(Context context, String fileName, String content) throws Exception {
        File dir = context.getExternalFilesDir(null);
        if (dir == null) return null;
        File file = new File(dir, fileName);
        Files.writeString(file.toPath(), content, StandardCharsets.UTF_8);
        return file.getAbsolutePath();
    }
    public static String read(Context context, String fileName) throws Exception {
        File dir = context.getExternalFilesDir(null);
        if (dir == null) return null;
        File file = new File(dir, fileName);
        if (!file.exists()) return null;
        return Files.readString(file.toPath(), StandardCharsets.UTF_8);
    }
}
