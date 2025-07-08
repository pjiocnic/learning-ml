To make your **Spring Boot application** read both:

* `application.properties` bundled **inside the JAR**, and
* an **external properties file** (outside the JAR) that overrides the internal one,

you can follow this approach:

---

### ✅ 1. **External File Location (Outside JAR)**

Place your external `application.properties` (or `application.yml`) in a known location outside the JAR, e.g.:

```
/opt/myapp/config/application.properties
```

---

### ✅ 2. **Launch Command with `--spring.config.location`**

When starting your JAR, pass the external config file path using:

```bash
java -jar myapp.jar --spring.config.location=file:/opt/myapp/config/
```

This will load external properties from that directory, and **merge them** with the ones inside the JAR.
👉 **External values will override internal ones.**

---

### 🔁 3. **OR Use `SPRING_CONFIG_LOCATION` Environment Variable**

Instead of passing it in the command line, you can also set an environment variable:

```bash
export SPRING_CONFIG_LOCATION=file:/opt/myapp/config/
java -jar myapp.jar
```

---

### 🧠 How It Works (Spring Boot Property Resolution Order)

Spring Boot loads config files in the following order (higher wins):

| Priority | Location                                                              |
| -------- | --------------------------------------------------------------------- |
| 1️⃣      | Command line args (`--spring.config.location=...`)                    |
| 2️⃣      | `SPRING_CONFIG_LOCATION` env var                                      |
| 3️⃣      | `application.properties` inside `src/main/resources` (bundled in JAR) |

---

### 🧪 Example: Inside JAR and Outside File

**Inside JAR (`src/main/resources/application.properties`)**

```properties
app.greeting=Hello from JAR
```

**Outside JAR (`/opt/myapp/config/application.properties`)**

```properties
app.greeting=Hello from external file
```

**Result:** when you run with `--spring.config.location=file:/opt/myapp/config/`, Spring will use:

```
app.greeting = Hello from external file
```

---

### 🛠 Bonus: Allowing Both `.properties` and `.yml`

Spring supports `.properties` and `.yml` in the same directory. So you can provide overrides in either format.

---

Would you like me to generate a sample runnable structure or Dockerfile setup for this pattern?
