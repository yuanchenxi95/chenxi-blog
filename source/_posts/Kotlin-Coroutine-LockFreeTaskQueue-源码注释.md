---
title: Kotlin Coroutine LockFreeTaskQueue 源码注释
date: 2021-11-09 20:03:21
tags:
---

## [Lock Free Task Queue](https://github.com/Kotlin/kotlinx.coroutines/blob/5e870c11311b3bfd58f0557c651f1d877e9e5827/kotlinx-coroutines-core/common/src/internal/LockFreeTaskQueue.kt)
对 [Kotlin Coroutine](https://github.com/Kotlin/kotlinx.coroutines) 的Lock Free Task Queue的注释

### 常量
```kotlin
    @Suppress("PrivatePropertyName", "MemberVisibilityCanBePrivate")
    internal companion object {
        const val INITIAL_CAPACITY = 8

        // 因为LockFreeTaskQueue用一个Atomic Long存储queue的state，来确保每次更新state都是原子操作
        // - head: 30 bit，用来记录circular queue的head index
        // - tail: 30 bit，用来记录circular queue的tail index
        // - frozen: 1 bit，indicate当前的queue是否被冻结
        // - closed: 1 bit, indicate当前的queue是否被close
        const val CAPACITY_BITS = 30
        
        const val MAX_CAPACITY_MASK = (1 shl CAPACITY_BITS) - 1
        const val HEAD_SHIFT = 0
        
        // Head Mask: 用来从Long state里获取head的Int值
        const val HEAD_MASK = MAX_CAPACITY_MASK.toLong() shl HEAD_SHIFT
        const val TAIL_SHIFT = HEAD_SHIFT + CAPACITY_BITS
        
        // Tail Mask: 用来从Long state里获取tail的Int值
        const val TAIL_MASK = MAX_CAPACITY_MASK.toLong() shl TAIL_SHIFT
        
        const val FROZEN_SHIFT = TAIL_SHIFT + CAPACITY_BITS
        
        // Frozen Mask: 用来从Long state里获取frozen的flag 
        const val FROZEN_MASK = 1L shl FROZEN_SHIFT
        const val CLOSED_SHIFT = FROZEN_SHIFT + 1
        
        // Closed Mask: 用来从Long state里获取closed的flag
        const val CLOSED_MASK = 1L shl CLOSED_SHIFT

        const val MIN_ADD_SPIN_CAPACITY = 1024

        @JvmField val REMOVE_FROZEN = Symbol("REMOVE_FROZEN")

        // Add to queue的enum状态
        
        // 成功Add to the queue
        const val ADD_SUCCESS = 0
        
        // 失败：queue已经为frozen状态
        const val ADD_FROZEN = 1
        
        // 失败：queue已经被close
        const val ADD_CLOSED = 2

        infix fun Long.wo(other: Long) = this and other.inv()
        
        // (this wo HEAD_MASK): 抹除Long state里head的值
        // (newHead.toLong() shl HEAD_SHIFT): 获取一个新的Long，并写入新的head值
        fun Long.updateHead(newHead: Int) = (this wo HEAD_MASK) or (newHead.toLong() shl HEAD_SHIFT)
        
        // (this wo TAIL_MASK): 抹除Long state里tail的值
        // (newHead.toLong() shl HEAD_SHIFT): 获取一个新的Long，并写入新的tail值
        fun Long.updateTail(newTail: Int) = (this wo TAIL_MASK) or (newTail.toLong() shl TAIL_SHIFT)
        
        inline fun <T> Long.withState(block: (head: Int, tail: Int) -> T): T {
            // 从Long state读出head值，并转化为Int
            val head = ((this and HEAD_MASK) shr HEAD_SHIFT).toInt()
            
            // 从Long state读出tail值，并转化为Int
            val tail = ((this and TAIL_MASK) shr TAIL_SHIFT).toInt()
            
            return block(head, tail)
        }

        // 获取state里的失败原因，如果是Close返回ADD_CLOSED状态，否则返回ADD_FROZEN状态
        // FROZEN | CLOSED
        fun Long.addFailReason(): Int = if (this and CLOSED_MASK != 0L) ADD_CLOSED else ADD_FROZEN
    }
```

### Internal变量
```kotlin
private typealias Core<E> = LockFreeTaskQueueCore<E>

internal class LockFreeTaskQueueCore<E : Any>(
    // Queue最大存储的值
    private val capacity: Int,
    
    // Indicate这个queue是否只有一个consumer，这个field非常重要
    // note 2 翻译：
    // 当这个queue有多个consumer的时候，这个queue并不是Lock Free
    // consumer spins直到producer完成它的操作
    private val singleConsumer: Boolean // true when there is only a single consumer (slightly faster)
) {
    // Mask: 用来获取queue的实际head index和tail index
    private val mask = capacity - 1
    
    // 在queue扩容时，会将新生成的queue存储在这个Atomic Ref里边
    // 如果在扩容期间，有别的thread对当前的queue进行操作，在操作完成时，会检查当前的queue的next是否为空
    // 如果是null，证明此时没有扩容，写入的atomic Long state成功，queue里的item操作也成功
    // 如果不为null，需要将状态更新到新生成的queue中，也就是next ref的queue，
    // 更新完成后，需要再次检查next的next是否为空，如果不为空，则说明queue再次扩容，
    // 需要重新进行更新，直到更新完成后的next为null。
    private val _next = atomic<Core<E>?>(null)
    
    // Atomic Long state，用来记录queue的状态，包括：head index，tail index，是否frozen，是否closed
    private val _state = atomic(0L)
    
    // Circular Queue array，生成一个长度为capacity的array，每个item都是一个atomic ref到null
    // 这个queue里边存储的是需要执行的任务，consumer会从queue的head读任务出来，producer会将任务加到tail的位置
    private val array = atomicArrayOfNulls<Any?>(capacity)
    
    // ...省略代码 
}
```

### 状态方法
```kotlin
    // 当head index == tail index的时候，queue为空
    // Note: it is not atomic w.r.t. remove operation (remove can transiently fail when isEmpty is false)
    val isEmpty: Boolean get() = _state.value.withState { head, tail -> head == tail }

    // tail index - head index，然后mask by MAX_CAPACITY_MASK为当前queue的size
    val size: Int get() = _state.value.withState { head, tail -> (tail - head) and MAX_CAPACITY_MASK }

    // 当前queue是否未关闭状态
    fun close(): Boolean {
        _state.update { state ->
            if (state and CLOSED_MASK != 0L) return true // ok - already closed
            if (state and FROZEN_MASK != 0L) return false // frozen -- try next
            state or CLOSED_MASK // try set closed bit
        }
        return true
    }
```

### Placeholder类，帮助标记状态的一个内部类
```kotlin
    // 当扩容时，如果发现array里边有null，说明有别的线程更新了tail index，但还没有写入element，这个时候扩容线程会将array里这个slot
    // 设置成Placeholder，来给其他线程在扩容后补写这个element
    // Instance of this class is placed into array when we have to copy array, but addLast is in progress --
    // it had already reserved a slot in the array (with null) and have not yet put its value there.
    // Placeholder keeps the actual index (not masked) to distinguish placeholders on different wraparounds of array
    // Internal because of inlining
    internal class Placeholder(@JvmField val index: Int)
```


### addLast - producer添加任务
```kotlin

    // ADD_CLOSED | ADD_FROZEN | ADD_SUCCESS
    fun addLast(element: E): Int {
        _state.loop { state ->
            // 如果当前状态为Frozen或是closed则返回失败原因
            if (state and (FROZEN_MASK or CLOSED_MASK) != 0L) return state.addFailReason() // cannot add
            
            state.withState { head, tail ->
                val mask = this.mask // manually move instance field to local for performance
                
                // 当queue只剩一个空位的时候，freeze并且扩容，因为single consumer的时候，可能会有一个element我们并不能overwrite
                // 在removeFirst的时候，因为removeFirst是先更新head index，再拿掉当前的element，所以需要进行extra margin of one element检查
                // If queue is Single-Consumer then there could be one element beyond head that we cannot overwrite,
                // so we check for full queue with an extra margin of one element
                if ((tail + 2) and mask == head and mask) return ADD_FROZEN // overfull, so do freeze & copy
                
                // If queue is Multi-Consumer then the consumer could still have not cleared element
                // despite the above check for one free slot.
                if (!singleConsumer && array[tail and mask].value != null) {
                    // 如果是multi consumer，这个时候要两种情况
                    // 1. 当queue的capacity < 1024或整个queue已经用掉了一半的时候，进行扩容
                    // 2. 否则，spin来等待consumer来拿任务
                    // 这个case就是之前singleConsumer注释所说的，这个queue并不是lock free，因为这种情况会spin来等待consumer来take这个任务
                    // There are two options in this situation
                    // 1. Spin-wait until consumer clears the slot
                    // 2. Freeze & resize to avoid spinning
                    // We use heuristic here to avoid memory-overallocation
                    // Freeze & reallocate when queue is small or more than half of the queue is used
                    if (capacity < MIN_ADD_SPIN_CAPACITY || (tail - head) and MAX_CAPACITY_MASK > capacity shr 1) {
                        return ADD_FROZEN
                    }
                    // otherwise spin
                    return@loop
                }
                
                // 获取新的tail的位置
                val newTail = (tail + 1) and MAX_CAPACITY_MASK
                
                // 原子操作，来确保state成功更新，如果compareAndSet返回false，此时state被别的thread更新，则继续loop
                // 因为这个library是target multi platform，这个地方的compareAndSet不止是Java的compareAndSet，也有JS和native
                if (_state.compareAndSet(state, state.updateTail(newTail))) {
                    
                    // successfully added
                    array[tail and mask].value = element
                    
                    // 因为在set value的时候，有可能别的thread更新了状态，所以需要进行检查
                    // could have been frozen & copied before this item was set -- correct it by filling placeholder
                    var cur = this
                    while(true) {
                        // 没有frozen，那说明addLast成功执行，exit loop
                        if (cur._state.value and FROZEN_MASK == 0L) break // all fine -- not frozen yet
                        
                        // 如果当前的queue被frozen了，证明在当前thread set queue的时候，这个queue被扩容了
                        // 之前注释next的时候提到过，queue扩容时会生成一个新的queue object，把当前的queue._next指到queue上
                        // 因为在扩容时，新生成的queue array会判断每个element是否为null，如果不为null则copy，
                        // 如果为null，扩容线程知道有别的thread更新了tail index但还没来得及写入新的element进array里边
                        
                        // 如果fillPlaceholder返回null，我们不需要再继续检查了，因为确认写入的element被成功复制到了新的queue里
                        // 如果返回的是新的queue，那么就说明，扩容线程复制的时候并没有复制当前element，fillPlaceholder进行了补写操作
                        // 当前element到扩容的array里边，这个时候，我们需要继续loop，因为可能扩容线程扩容之后，又有别的线程进行了扩容
                        // 这时loop会持续补写直到我们发现不需要array里的值不是Placeholder instance为止
                        cur = cur.next().fillPlaceholder(tail, element) ?: break
                    }
                    return ADD_SUCCESS // added successfully
                }
            }
        }
    }

    private fun fillPlaceholder(index: Int, element: E): Core<E>? {
        // 获取这个index里边存的值
        val old = array[index and mask].value
        /*
         * addLast actions: addLast的操作
         * 1) Commit tail slot 更新tail的index，也就是allocate tail的slot
         * 2) Write element to array slot 写入element到array里边
         * 3) Check for array copy 检查array是否扩容
         *
         * 在操作过程中如果第2步和第3步过程中，发生了扩容，consumer有可能已经获取了这个element
         * If copy happened between 2 and 3 then the consumer might have consumed our element,
         * then another producer might have written its placeholder in our slot, so we should
         * perform *unique* check that current placeholder is our to avoid overwriting another producer placeholder
         * 如果Placeholder里边的index是方法传入的index，这时我们确定这个Placeholder是当前线程的Placeholder，而不是其他producer的
         */
        if (old is Placeholder && old.index == index) {
            // 因为扩容的时候，扩容线程复制了老的array，此时，当前producer线程并没有写入新的element，这个操作是把element写进扩容后的array
            array[index and mask].value = element
            // we've corrected missing element, should check if that propagated to further copies, just in case
            return this
        }
        // 返回null，这种情况是扩容线程在扩容的时候，已经看到了producer线程写入的新element
        // 这个时候我们不需要做进行next的check，因为之后即便再有扩容，已经写进当前扩容的element都会被复制到之后的扩容array里
        // it is Ok, no need for further action
        return null
    }
```

### removeFistOrNull consumer来拿任务，如果没有任务，则返回null
```kotlin

    // REMOVE_FROZEN | null (EMPTY) | E (SUCCESS)
    fun removeFirstOrNull(): Any? {
        _state.loop { state ->
            // 当前queue已被frozen，发生了扩容，返回这个状态，让调用这个方法的consumer决定该做什么
            if (state and FROZEN_MASK != 0L) return REMOVE_FROZEN // frozen -- cannot modify
            state.withState { head, tail ->
                // 当前queue为空，返回null
                if ((tail and mask) == (head and mask)) return null // empty
                val element = array[head and mask].value
                if (element == null) {
                    // 如果是single consumer，则producer没有完成加入element
                    // If queue is Single-Consumer, then element == null only when add has not finished yet
                    if (singleConsumer) return null // consider it not added yet
                    // 如果是multi consumer，我们需要spin，来尝试继续获取element，这个是之前说的这个queue会spin
                    // retry (spin) until consumer adds it
                    return@loop
                }

                // 如果当前element是Placeholder，说明producer还没来得及补写扩容之后的queue array，这时我们认为状态是not added yet，所以返回null
                // element == Placeholder can only be when add has not finished yet
                if (element is Placeholder) return null // consider it not added yet

                // 我们这个地方不能直接更新element为null，因为会有一个edge case
                // 情况为：当前有两个线程，一个producer线程即将进行扩容，一个consumer线程，
                // 假设consumer线程直接把array[head]变成了null，之后producer线程进行扩容，因为扩容时，null的element会被当做是别的producer线程还没来得及写入的element
                // 这时扩容线程把这个element变成了placeholder，这回break queue的状态

                // 正确操作是先更新head index，如果更新成功，再将array[previous head]设成null，因为这个时候即便producer线程进行扩容，也只会copy新的head到tail的elements

                // we cannot put null into array here, because copying thread could replace it with Placeholder and that is a disaster
                val newHead = (head + 1) and MAX_CAPACITY_MASK
                if (_state.compareAndSet(state, state.updateHead(newHead))) {
                    // Array could have been copied by another thread and it is perfectly fine, since only elements
                    // between head and tail were copied and there are no extra steps we should take here
                    array[head and mask].value = null // now can safely put null (state was updated)
                    return element // successfully removed in fast-path
                }
                // multi consumer的时候需要spin来确保当前head不被别的consumer拿掉
                // Multi-Consumer queue must retry this loop on CAS failure (another consumer might have removed element)
                if (!singleConsumer) return@loop
                
                // 如果是single consumer的case，之前的compareAndSet失败是因为producer线程进行了扩容
                // 这个时候需要从扩容之后的queue里边拿到element，并且更新head
                // Single-consumer queue goes to slow-path for remove in case of interference
                var cur = this
                while (true) {
                    @Suppress("UNUSED_VALUE")
                    cur = cur.removeSlowPath(head, newHead) ?: return element
                }
            }
        }
    }

    private fun removeSlowPath(oldHead: Int, newHead: Int): Core<E>? {
        _state.loop { state ->
            state.withState { head, _ ->
                // Extra检查，head值不应该变，如果变化了，说明并不是single consumer，
                // 这个时候会报错，因为single consumer的操作和multi consumer操作不一样，assumption不正确queue的操作就不正确
                assert { head == oldHead } // "This queue can have only one consumer"
                
                // Frozen状态说明扩容了，这个时候要去point到扩容之后的queue
                if (state and FROZEN_MASK != 0L) {
                    // state was already frozen, so removed element was copied to next
                    return next() // continue to correct head in next
                }
                // 原子操作更新head，如果这个时候更新成功，则将head 设成null
                // 更新不成功，说明有别的producer线程又进行了扩容，这个时候需要继续loop，直到head成功被更新，也就是没有别的线程扩容
                if (_state.compareAndSet(state, state.updateHead(newHead))) {
                    array[head and mask].value = null // now can safely put null (state was updated)
                    return null
                }
            }
        }
    }
```

### 扩容逻辑，在addLast的时候，满足之前提到的条件会进行扩容
扩容实际是copy之前的array里边的元素到新的array里边
```kotlin
    // 当前queue的next或是copy，如果有next则说明别的线程扩容了，如果next是null，则进行扩容
    fun next(): LockFreeTaskQueueCore<E> = allocateOrGetNextCopy(markFrozen())

    // 把当前state标记成frozen，提示其他线程，这个queue已经过期（有新的扩容queue生成）
    private fun markFrozen(): Long =
        _state.updateAndGet { state ->
            if (state and FROZEN_MASK != 0L) return state // already marked
            state or FROZEN_MASK
        }

    private fun allocateOrGetNextCopy(state: Long): Core<E> {
        _next.loop { next ->
            // 如果next不为null，期间已有别的线程扩容了
            if (next != null) return next // already allocated & copied
            // 如果是null，则进行扩容
            // 因为在扩容时，可能有别的线程更改了状态，所以compareAndSet失败的时候，next依旧为null，这时进行继续loop check
            // 这个地方也是个spin
            _next.compareAndSet(null, allocateNextCopy(state))
        }
    }

    private fun allocateNextCopy(state: Long): Core<E> {
        // 生成一个新的Queue，容量是之前的两倍
        val next = LockFreeTaskQueueCore<E>(capacity * 2, singleConsumer)
        state.withState { head, tail ->
            var index = head
            // 扩容逻辑，range是head index到tail index
            // 之前提到过，removeFirst会先更新head index，addLast会先更新tail index
            while (index and mask != tail and mask) {
                // 如果array[index]不为null，则复制到新的array，如果是null，说明此时producer线程更新完tail index后还没写入array
                // 这个时候我们创建一个Placeholder，来给producer线程补写element进扩容后的array
                // replace nulls with placeholders on copy
                val value = array[index and mask].value ?: Placeholder(index)
                next.array[index and next.mask].value = value
                index++
            }
            // 更新状态，reset frozen flag为0
            next._state.value = state wo FROZEN_MASK
        }
        return next
    }
```

### 最后LockFreeTaskQueue，这里边的逻辑比较straight forward
```kotlin
internal open class LockFreeTaskQueue<E : Any>(
    singleConsumer: Boolean // true when there is only a single consumer (slightly faster & lock-free)
) {
    private val _cur = atomic(Core<E>(Core.INITIAL_CAPACITY, singleConsumer))

    // Note: it is not atomic w.r.t. remove operation (remove can transiently fail when isEmpty is false)
    val isEmpty: Boolean get() = _cur.value.isEmpty
    val size: Int get() = _cur.value.size

    fun close() {
        // 不断尝试close当前queue，因为可能发生扩容，所以要不断spin来尝试
        _cur.loop { cur ->
            if (cur.close()) return // closed this copy
            _cur.compareAndSet(cur, cur.next()) // move to next
        }
    }

    fun addLast(element: E): Boolean {
        _cur.loop { cur ->
            // addLast
            when (cur.addLast(element)) {
                Core.ADD_SUCCESS -> return true
                Core.ADD_CLOSED -> return false
                // Frozen状态说明有扩容，获取扩容后的queue，这个地方进行了spin
                Core.ADD_FROZEN -> _cur.compareAndSet(cur, cur.next()) // move to next
            }
        }
    }

    @Suppress("UNCHECKED_CAST")
    fun removeFirstOrNull(): E? {
        _cur.loop { cur ->
            val result = cur.removeFirstOrNull()
            if (result !== Core.REMOVE_FROZEN) return result as E?
            // Frozen状态说明有扩容，获取扩容后的queue，这个地方进行了spin
            _cur.compareAndSet(cur, cur.next())
        }
    }

    // Used for validation in tests only
    fun <R> map(transform: (E) -> R): List<R> = _cur.value.map(transform)

    // Used for validation in tests only
    fun isClosed(): Boolean = _cur.value.isClosed()
}
```

### 总结
基本每一个要处理异步编程的library都需要有个queue来维护任务，Kotlin这个coroutine因为要target multiplatform，
写这么个queue要考虑很多case，因为有的platform可能是single consumer，有的又是multi consumer，
为了这个还写了不同的处理逻辑，也算是做到极致了。

对比别的library可能值需要做到MPSC，比如Reactor的 [MpscLinkedQueue](https://github.com/reactor/reactor-core/blob/933fc90572194db080590e3b2f96b91147aebb4a/reactor-core/src/main/java/reactor/util/concurrent/MpscLinkedQueue.java)
，kotlin coroutine的queue的逻辑要复杂得多。

在效率方面，如果是对于target是JVM的，应该跟用线程池实现NIO的效率差不多，我记得之前看扔物线的 [Kotlin的协程](https://youtu.be/LKsGkEsEE7w)
提到过这个概念，Kotlin的coroutine并不会比线程池更效率，因为它底层也是这么实现的。

而且从另一个角度看，在JS里，这种在Kotlin维护一个task queue，未必比native的event loop更效率，
因为Kotlin最终是transpile成了JS，而Node里边的event loop是libuv用C实现的。

不知道这个想法对不对。
