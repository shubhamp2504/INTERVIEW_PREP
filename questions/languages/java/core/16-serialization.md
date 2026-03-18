# ☕ Core Java — Serialization (Q1–Q8)

> **Source**: 240 Core Java Interview Questions PDF  
> **Coverage**: Serialization basics, Serializable interface, SerialVersionUID, alternatives, reference serialization

---

<a id="q1"></a>
## Q1. What is serialization in Java?

### 📝 One-Liner
Serialization converts an object's state into a **byte stream** for storage or transmission — deserialization reverses it.

### 🔑 Quick Answer
Serialization = converting object → bytes → file/network/DB. The byte stream includes: object data, object type, stored data types. Use `ObjectOutputStream.writeObject()` to serialize and `ObjectInputStream.readObject()` to deserialize. Implemented via `java.io.Serializable` marker interface. *(Serialization = object ko bytes me convert karo taaki store ya transfer ho sake)*

### 📖 How It Works

```
Serialization Flow:
┌──────────┐   serialize   ┌───────────┐
│  Object   │ ────────────→│ Byte Stream│ → File/Network/DB
│ (in memory)│             │            │
└──────────┘   deserialize ┌───────────┐
                ←───────── │ Byte Stream│
                            └───────────┘
```

### ⚡ Remember
`Serialization = object → bytes | Deserialization = bytes → object | ObjectOutputStream/InputStream`

---

<a id="q2"></a>
## Q2. What is the main purpose of serialization?

### 📝 One-Liner
Persistence, network communication, object copying, and distributing objects across JVMs.

### 🔑 Quick Answer
Main uses: (1) **Persistence** — save object state to file/DB, recreate later. (2) **Communication** — pass objects over network (RPC, RMI). (3) **Copying** — deep copy via serialize + deserialize (byte array copy). (4) **Distribution** — share objects across different JVMs or distributed caches. *(Serialization ka use: save, transfer, copy, distribute)*

### ⚡ Remember
`Purpose: Persistence + Communication + Copying + Distribution across JVMs`

---

<a id="q3"></a>
## Q3. What are alternatives to Java serialization?

### 📝 One-Liner
**JSON** (Jackson/Gson), **XML** (JAXB/JIBX), **Protocol Buffers**, **Avro**, **Kryo** — all more portable and efficient.

### 🔑 Quick Answer
(1) **JSON** (Jackson, Gson) — human-readable, language-agnostic, widely used in REST APIs. (2) **XML** (JAXB, JIBX) — marshall to XML, unmarshal back. (3) **Protocol Buffers** (Google) — binary, fast, schema-defined. (4) **Avro** — schema evolution, big data. (5) **Kryo** — fast Java binary serialization. Java native serialization has known security issues and is being phased out. *(JSON/Protobuf preferred over Java serialization — faster, safer, portable)*

### ⚡ Remember
`JSON (Jackson) > XML (JAXB) > Protobuf > Avro | Java serialization has security issues`

---

<a id="q4"></a>
## Q4. Explain Serializable interface?

### 📝 One-Liner
`Serializable` is a **marker interface** (no methods) in `java.io` — implementing it signals to JVM that instances can be serialized.

### 🔑 Quick Answer
`public interface Serializable { }` — completely empty. Just implementing it tells the JVM: "this class's objects can be converted to bytes." Without it, serialization throws `NotSerializableException`. All subclasses of a Serializable class are also serializable. *(Serializable = marker interface, koi method nahi, sirf signal hai ki serialize ho sakta hai)*

### ⚡ Remember
`Serializable = marker interface | No methods | Signals JVM | NotSerializableException without it`

---

<a id="q5"></a>
## Q5. How to make an object serializable?

### 📝 One-Liner
Implement `Serializable`, use `ObjectOutputStream` to write and `ObjectInputStream` to read.

### 🔑 Quick Answer
Steps: (1) Class implements `Serializable`. (2) All referenced objects must also be Serializable (or marked `transient`). (3) Write with `ObjectOutputStream.writeObject(obj)`. (4) Read with `ObjectInputStream.readObject()`. *(3 steps: implements Serializable, ObjectOutputStream se likho, ObjectInputStream se padho)*

### 💻 Code Example

```java
// 1. Implement Serializable
public class Employee implements Serializable {
    private String name;
    private int age;
    private transient String password;  // won't serialize
}

// 2. Serialize
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("emp.ser"))) {
    oos.writeObject(new Employee("Alice", 30, "secret"));
}

// 3. Deserialize
try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("emp.ser"))) {
    Employee emp = (Employee) ois.readObject();
    // emp.password will be null (transient)
}
```

### ⚡ Remember
`implements Serializable | ObjectOutputStream.writeObject | ObjectInputStream.readObject`

---

<a id="q6"></a>
## Q6. What is SerialVersionUID and its importance?

### 📝 One-Liner
A unique 64-bit `long` identifier that ensures the **sender and receiver** of a serialized object have compatible class versions.

### 🔑 Quick Answer
`private static final long serialVersionUID = 1L;`. When serializing, the SUID is written with the object. On deserialization, JVM compares the stored SUID with the loaded class's SUID. Mismatch → `InvalidClassException`. If not defined, JVM generates one from class structure (fragile — minor changes break it). Always define explicitly. *(SUID = class version check — sender aur receiver ki class compatible hai ya nahi)*

### 💻 Code Example

```java
public class Employee implements Serializable {
    private static final long serialVersionUID = 1L;  // ⭐ always define explicitly
    private String name;
    private int age;
}
```

### ⚡ Remember
`SUID = version check | Mismatch → InvalidClassException | Always define explicitly`

---

<a id="q7"></a>
## Q7. What happens if we don't define SerialVersionUID?

### 📝 One-Liner
JVM auto-generates one from class structure — any class change (even adding a method) may **break deserialization**.

### 🔑 Quick Answer
Without explicit SUID, JVM computes a hash from: class name, interfaces, fields, methods. Any modification changes the hash → old serialized data becomes incompatible → `InvalidClassException`. Performance impact: JVM computes hash at runtime. Best practice: **always define SUID explicitly**. *(SUID define nahi karoge toh class me chhota sa change bhi purani files tod dega)*

### ⚡ Remember
`No SUID → JVM generates → any change breaks deserialization | Always define it!`

---

<a id="q8"></a>
## Q8. When we serialize an object, are its references also serialized?

### 📝 One-Liner
Yes — all referenced objects must also implement `Serializable`, otherwise `NotSerializableException` is thrown.

### 🔑 Quick Answer
When you serialize an object, the serialization mechanism traverses all its references and serializes them too (deep serialization). Every referenced object must implement `Serializable`. If any reference is not serializable, mark it `transient` to skip it. Circular references are handled by Java's serialization mechanism (uses reference tracking). *(Haan, sabhi referenced objects bhi serialize hote hain — sab Serializable hone chahiye ya transient mark karo)*

### ⚡ Remember
`References also serialized | All must be Serializable | Non-serializable → mark transient | Circular refs handled`
