---
title: Java屏蔽序列化版本不一致异常
date: 2025-04-28 19:02:45
tags: 
  - java
---

经常会碰到实现了序列化接口，但是忘记添加序列化id，导致线上的数据无法被正常的反序列化

使用CompatibleInputStream代替ObjectInputStream

```java
/**
 * 处理 jdk 序列化反序列化时，uid不一样的情况
 */
@Slf4j
public class CompatibleInputStream extends ObjectInputStream {
	public CompatibleInputStream(InputStream in) throws IOException {
		super(in);
	}


	@Override
	protected ObjectStreamClass readClassDescriptor() throws IOException, ClassNotFoundException {
		ObjectStreamClass resultClassDescriptor = super.readClassDescriptor(); // initially streams descriptor
		Class localClass; // the class in the local JVM that this descriptor represents.
		try {
			localClass = Class.forName(resultClassDescriptor.getName());
		} catch (ClassNotFoundException e) {
			return resultClassDescriptor;
		}
		ObjectStreamClass localClassDescriptor = ObjectStreamClass.lookup(localClass);
		if (localClassDescriptor != null) { // only if class implements serializable
			final long localSUID = localClassDescriptor.getSerialVersionUID();
			final long streamSUID = resultClassDescriptor.getSerialVersionUID();
			if (streamSUID != localSUID) { // check for serialVersionUID mismatch.
				log.warn("Overriding serialized class version mismatch: local serialVersionUID = [{}] stream serialVersionUID = [{}]", localSUID, streamSUID);
				resultClassDescriptor = localClassDescriptor; // Use local class descriptor for deserialization
			}
		}
		return resultClassDescriptor;
	}
}
```