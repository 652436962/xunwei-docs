# 第096章  kobject释放实例分析实验

本章分析kobject 的释放过程。
## 96.1 Kobj 是如何释放的？
在Linux 内核中，kobject 的生命周期管理通过引用计数kref 实现。释放kobject 使用kobject_put()函数。kobject_put()函数定义在lib/kobject.c 文件中，函数实现如代码清单96.1 所示。代码清单96.1
```c
void kobject_put(struct kobject *kobj)
{
  if (kobj) {
    if (!kobj->state_initialized)
      WARN(1, KERN_WARNING "kobject: '%s' (%p): is not initialized, yet kobject_put() is being called.\n",
      kobject_name(kobj), kobj);
    kref_put(&kobj->kref, kobject_release);
  }
}
EXPORT_SYMBOL(kobject_put);
```
通过代码清单96.1 第7 行可知，kobject_put()函数调用了内核通用的引用计数管理函数kref_put()函数来释放引用，并设置了释放的回调函数kobejct_release()，当引用计数为0 时，kobkect_release()就会被调用。kobject_release 的具体实现如代码清单96.2 所示。代码清单96.2
```c
static void kobject_release(struct kref *kref)
{
    struct kobject *kobj = container_of(kref, struct kobject, kref);
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
    unsigned long delay = HZ + HZ * (get_random_int() & 0x3);
    pr_info("kobject: '%s' (%p): %s, parent %p (delayed %ld)\n",
         kobject_name(kobj), kobj, __func__, kobj->parent, delay);
    INIT_DELAYED_WORK(&kobj->release, kobject_delayed_cleanup);
    schedule_delayed_work(&kobj->release, delay);
#else
    kobject_cleanup(kobj);
#endif }
```
通过代码清单96.2 可知，kobject_release()函数最终会去调用kobject_cleanup() 函数，

kobject_cleanup()函数实现如代码清单96.3 所示。代码清单96.3
```c
static void kobject_cleanup(struct kobject *kobj)
{
     struct kobj_type *t = get_ktype(kobj);
    const char *name = kobj->name;
    pr_debug("kobject: '%s' (%p): %s, parent %p\n", kobject_name(kobj), kobj, __func__, kobj->parent);
    if (t && !t->release)
        pr_debug("kobject: '%s' (%p): does not have a release() function, it is broken and must be fixed.\n",
                kobject_name(kobj), kobj);
        /* send "remove" if the caller did not do it but sent "add" */
         if (kobj->state_add_uevent_sent && !kobj->state_remove_uevent_sent) {
             pr_debug("kobject: '%s' (%p): auto cleanup 'remove' event\n", kobject_name(kobj), kobj);
             kobject_uevent(kobj, KOBJ_REMOVE); }
        /* remove from sysfs if the caller did not do it */
         if (kobj->state_in_sysfs) {
             pr_debug("kobject: '%s' (%p): auto cleanup kobject_del\n", kobject_name(kobj), kobj);
            kobject_del(kobj);
        }
         if (t && t->release) {
             pr_debug("kobject: '%s' (%p): calling ktype release\n", kobject_name(kobj), kobj);
             t->release(kobj);
         }
         /* free name if we allocated it */
        if (name) {
           pr_debug("kobject: '%s': free name\n", name);
          kfree_const(name);
        }
}
```
通过代码清单96.3 可知，第3 行获取kobject 的类型。kobj_type 是内核中描述kobject 类型的结构体，定义如代码清单96.4 所示。代码清单96.4
```c
struct kobj_type {
  void (*release)(struct kobject *kobj); //自定义释放函数
  const struct sysfs_ops *sysfs_ops;//sysfs 属性操作函数
  struct attribute **default_attrs;//默认sysfs 属性列表
  const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
  const void *(*namespace)(struct kobject *kobj);
  void (*get_ownership)(struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};
```

