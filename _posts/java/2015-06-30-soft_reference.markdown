---
layout: post
category: "java"
title:  "深入C++探讨下SoftReference类"
tags: [java,SoftReference,c++,c]
---

看过SoftReference类的java源码，是不是会疑惑clock的值是怎么来的？

找找C++源码。。。。。。(jdk版本: jdk9-b66)

hotspot/src/share/vm/memory/referenceProcessor.cpp

<pre class="prettyPrint">
void ReferenceProcessor::init_statics() {
  // We need a monotonically non-decreasing time in ms but
  // os::javaTimeMillis() does not guarantee monotonicity.
  jlong now = os::javaTimeNanos() / NANOSECS_PER_MILLISEC;

  // Initialize the soft ref timestamp clock.
  _soft_ref_timestamp_clock = now;
  // Also update the soft ref clock in j.l.r.SoftReference
  java_lang_ref_SoftReference::set_clock(_soft_ref_timestamp_clock);

  _always_clear_soft_ref_policy = new AlwaysClearPolicy();
  _default_soft_ref_policy      = new COMPILER2_PRESENT(LRUMaxHeapPolicy())
                                      NOT_COMPILER2(LRUCurrentHeapPolicy());
  if (_always_clear_soft_ref_policy == NULL || _default_soft_ref_policy == NULL) {
    vm_exit_during_initialization("Could not allocate reference policy object");
  }
  guarantee(RefDiscoveryPolicy == ReferenceBasedDiscovery ||
            RefDiscoveryPolicy == ReferentBasedDiscovery,
            "Unrecognized RefDiscoveryPolicy");
}
</pre>

此处将clock设置为当前时间。

此文件中有个函数process\_discovered\_references

<pre class="prettyPrint">
ReferenceProcessorStats ReferenceProcessor::process_discovered_references(
  BoolObjectClosure*           is_alive,
  OopClosure*                  keep_alive,
  VoidClosure*                 complete_gc,
  AbstractRefProcTaskExecutor* task_executor,
  GCTimer*                     gc_timer,
  GCId                         gc_id) {
  NOT_PRODUCT(verify_ok_to_handle_reflists());

  assert(!enqueuing_is_done(), "If here enqueuing should not be complete");
  // Stop treating discovered references specially.
  disable_discovery();

  // If discovery was concurrent, someone could have modified
  // the value of the static field in the j.l.r.SoftReference
  // class that holds the soft reference timestamp clock using
  // reflection or Unsafe between when discovery was enabled and
  // now. Unconditionally update the static field in ReferenceProcessor
  // here so that we use the new value during processing of the
  // discovered soft refs.

  _soft_ref_timestamp_clock = java_lang_ref_SoftReference::clock();

  bool trace_time = PrintGCDetails && PrintReferenceGC;

  // Soft references
  size_t soft_count = 0;
  {
    GCTraceTime tt("SoftReference", trace_time, false, gc_timer, gc_id);
    soft_count =
      process_discovered_reflist(_discoveredSoftRefs, _current_soft_ref_policy, true,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }

  update_soft_ref_master_clock();

  // Weak references
  size_t weak_count = 0;
  {
    GCTraceTime tt("WeakReference", trace_time, false, gc_timer, gc_id);
    weak_count =
      process_discovered_reflist(_discoveredWeakRefs, NULL, true,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }

  // Final references
  size_t final_count = 0;
  {
    GCTraceTime tt("FinalReference", trace_time, false, gc_timer, gc_id);
    final_count =
      process_discovered_reflist(_discoveredFinalRefs, NULL, false,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }

  // Phantom references
  size_t phantom_count = 0;
  {
    GCTraceTime tt("PhantomReference", trace_time, false, gc_timer, gc_id);
    phantom_count =
      process_discovered_reflist(_discoveredPhantomRefs, NULL, false,
                                 is_alive, keep_alive, complete_gc, task_executor);

    // Process cleaners, but include them in phantom statistics.  We expect
    // Cleaner references to be temporary, and don't want to deal with
    // possible incompatibilities arising from making it more visible.
    phantom_count +=
      process_discovered_reflist(_discoveredCleanerRefs, NULL, true,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }

  // Weak global JNI references. It would make more sense (semantically) to
  // traverse these simultaneously with the regular weak references above, but
  // that is not how the JDK1.2 specification is. See #4126360. Native code can
  // thus use JNI weak references to circumvent the phantom references and
  // resurrect a "post-mortem" object.
  {
    GCTraceTime tt("JNI Weak Reference", trace_time, false, gc_timer, gc_id);
    if (task_executor != NULL) {
      task_executor->set_single_threaded_mode();
    }
    process_phaseJNI(is_alive, keep_alive, complete_gc);
  }

  return ReferenceProcessorStats(soft_count, weak_count, final_count, phantom_count);
}
</pre>


然后一步一步的执行到了 policy->should\_clear\_reference

Java提供了几种SoftReference策略

1. NeverClearPolicy (没被使用)
2. AlwaysClearPolicy
3. LRUCurrentHeapPolicy
4. LRUMaxHeapPolicy

这四种策略在文件jdk9/hotspot/src/share/vm/memory/referencePolicy.hpp中定义(代码这里就不贴了)

前两种没必要研究

3，4 在文件jdk9/hotspot/src/share/vm/memory/referencePolicy.cpp中

<pre class="prettyPrint">
#include "precompiled.hpp"
#include "classfile/javaClasses.hpp"
#include "memory/referencePolicy.hpp"
#include "memory/universe.hpp"
#include "runtime/arguments.hpp"
#include "runtime/globals.hpp"

LRUCurrentHeapPolicy::LRUCurrentHeapPolicy() {
  setup();
}

// Capture state (of-the-VM) information needed to evaluate the policy
void LRUCurrentHeapPolicy::setup() {
  _max_interval = (Universe::get_heap_free_at_last_gc() / M) * SoftRefLRUPolicyMSPerMB;
  assert(_max_interval >= 0,"Sanity check");
}

// The oop passed in is the SoftReference object, and not
// the object the SoftReference points to.
bool LRUCurrentHeapPolicy::should_clear_reference(oop p,
                                                  jlong timestamp_clock) {
  jlong interval = timestamp_clock - java_lang_ref_SoftReference::timestamp(p);
  assert(interval >= 0, "Sanity check");

  // The interval will be zero if the ref was accessed since the last scavenge/gc.
  if(interval <= _max_interval) {
    return false;
  }

  return true;
}

/////////////////////// MaxHeap //////////////////////

LRUMaxHeapPolicy::LRUMaxHeapPolicy() {
  setup();
}

// Capture state (of-the-VM) information needed to evaluate the policy
void LRUMaxHeapPolicy::setup() {
  size_t max_heap = MaxHeapSize;
  max_heap -= Universe::get_heap_used_at_last_gc();
  max_heap /= M;

  _max_interval = max_heap * SoftRefLRUPolicyMSPerMB;
  assert(_max_interval >= 0,"Sanity check");
}

// The oop passed in is the SoftReference object, and not
// the object the SoftReference points to.
bool LRUMaxHeapPolicy::should_clear_reference(oop p,
                                             jlong timestamp_clock) {
  jlong interval = timestamp_clock - java_lang_ref_SoftReference::timestamp(p);
  assert(interval >= 0, "Sanity check");

  // The interval will be zero if the ref was accessed since the last scavenge/gc.
  if(interval <= _max_interval) {
    return false;
  }

  return true;
}
</pre>

仔细看看发现

【下面写了一堆 还是删掉了，直接看代码吧，都能懂】

总结下：

存活时间大于\_max\_interval就回收

就是3，4策略对于\_max\_interval的计算方式不同

这里涉及到一个配置项 -XX:SoftRefLRUPolicyMSPerMB=1000

LRUCurrentHeapPolicy:当前堆可用空间，每兆1000毫秒

LRUMaxHeapPolicy: (当前堆大小-已使用的)

\_heap\_capacity\_at\_last\_gc - \_heap\_used\_at\_last\_gc

LRUMaxHeapPolicy: (最大堆大小-已使用的)

MaxHeapSize - \_heap\_used\_at\_last\_gc

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-如有错误请指正\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-