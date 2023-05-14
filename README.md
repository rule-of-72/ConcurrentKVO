# ConcurrentKVO

Classes that implement:

- A Reader/Writer Lock built using a GCD DispatchQueue.
- A KVO-compliant property.
- A concurrent, KVO-compliant property, which uses neither of the above.

Most implementations of KVO-compliant properties that use R/W locks for thread safety have a race condition.
- You can’t call `willChangeValue` and `didChangeValue` **inside** the barrier/write operation.
    - NSObject calls your property getter from within these calls, which causes a deadlock.
    - GCD detects this and throws an exception to terminate your app.
- But if you call `willChangeValue` and `didChangeValue` **outside** the barrier/write operation, you permit a race condition.
    - Other writers may interleave their KVO calls with yours, even though the write itself is protected.
    - This creates an “audit trail” of KVO notifications that doesn’t accurately reflect the true sequence of property value changes.

The `ConcurrentKVO` class prevents this race by sending the KVO notification from inside the barrier. It avoids deadlock by using thread-local storage to record the property value before sending the notification. The property getter checks for this, so it doesn’t need to re-enter the lock when it gets called on the same thread as a running barrier operation.

Although recursive locks are considered an anti-pattern, the NSObject implementation of KVO requires that the public getter be available for re-entrancy from within the setter, so this is unavoidable.

