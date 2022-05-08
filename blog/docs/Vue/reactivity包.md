---
title: reactive 原理
date: 2022-05-08
---

reactive 实现原理,目前 vue3.0支持单个API单独安装，所以我们可以单独安装这个API，实现过程 pnpm i @vue/reactivity -D

# reractivity 包下所包含API列表

1、以下 API  from  ref.ts

- ref           // 作为API导出使用的的 ref(val)
  - ref对象是可变的，当我们给 .value 赋新值。.value还是响应式的，
  - ref 定义 未使用，则不会收集依赖，只有 模版中使用到才会 收集依赖，收集以来函数放在 get() 函数中，（tree-shaking）

- shallowRef    // 作为API导出使用的的 shallowRef(val)
  - 与 ref 不同，这里相当于是浅拷贝。只有 shallowRef 中是响应式时。shallowRef通常用于大型数据结构的性能优化，或者外部状态系统的集成。能够深度避免转换

- isRef
- toRef
- toRefs
- unref
- proxyRefs
- customRef
- triggerRef
-
- Ref       // ts 类型定义｛ value, [RefSymbol] ｝
- ToRef
- ToRefs
- UnwrapRef
- ShallowRef
- ShallowUnwrapRef
- RefUnwrapBailTypes
- CustomRefFactory

2、以下 API from reactive.ts

- reactive
- readonly
- isReactive
- isReadonly
- isShallow
- isProxy
- shallowReactive
- shallowReadonly
- markRaw
- toRaw

- reactiveFlags
- DeepReadonly
- ShallowReactive
- UnwrapNestedRefs

3、以下API from computed

- computed
- ComputedRef
- WritableComputedRef
- WritableComputedOptions
- computedGetter
- ComputedSetter

4、以下 API from effect

- effect
- stop
- trigger
- track
- enableTracking
- pauseTracking
- resetTracking
- ITERATE_KEY
- ReactiveEffect
- ReactiveEffectRunner
- ReactiveEffectOptions
- EffectScheduler
- DebuggerOptions
- DebuggerEvent
- DebuggerEventExtraInfo

5、以下 API from effectScope

- effectScope
- EffectScope
- getCurrentScope
- onScopeDispose

6、 以下 API from deferredComputed

- deferredComputed

实现流程

## 1、ref

   ```vue
    function ref(val){
      return createRef(val)
    }   

    function createRef(val){
      if(isRef(val)){
        return value
      }
      // 这里 RefImpl (rawValue, shallow)
      return new RefImpl(val, false)
    }

    class RefImpl {
      constructor(value, shallow){
        this.__v__isShallow = shallow
        this.deep = undefined
        this.__v__isRef = true
        // toRaw()可以从reactive()、readonly()、shallowReactive()或shallowReadonly()创建的代理中返回原始对
        this._rawValue = shallow ? value : toRaw(val)
        this._value = shallow ? value : toReactive(value)
      }
      get(){
        //  收集依赖
        trackRefValue（this）
        return this._value
      }
      set(newVal){
        //  判断新值是不是对象，如果是对象则使用toRaw包裹
        //  TODO: __v_isShallow待确认 
        newVal = this.__v_isShallow ? newVal : toRaw(newVal)

        // 判断新旧值的改变
        if(hasChanged(newVal, this._rawValue)){
          <// 新旧值不同场景
          this._rawValue = newValue
          this._value = this.is_shallow ? newVal : toReactive(newVal)
          // 触发依赖，个人理解：新旧值变化重新触发依赖重新收集
          triggerRefValue(this, newVal)
        }
      }
    }

    function hasChanged(v1,v2){
      // 方法判断两个值是否为同一个值 
      // Object.is() 方法判断两个值是否为同一个值。如果满足以下条件则两个值相等: 
      //    都是 undefined
      //    都是 null
      //    都是 true 或 false
      //    都是相同长度的字符串且相同字符按相同顺序排列
      //    都是相同对象（意味着每个对象有同一个引用）
      //    都是数字且
      //      都是 +0
      //      都是 -0
      //      都是 NaN
      //      或都是非零而且非 NaN 且为同一个值

      return !Object.is(va,v2)
    }

    function triggerRefValue(ref){
      // prd
      trackEffect(ref.dep)
    }

    function trackEffect(dep){
      // 用 dep 来存放所有的 effect

     // TODO
     //  这里是一个优化点
     //  先看看这个依赖是不是已经收集了，
     //  已经收集的话，那么就不需要在收集一次了
     //   可能会影响 code path change 的情况
     //   需要每次都 cleanupEffect
     //   shouldTrack = !dep.has(activeEffect!);
    if (!dep.has(activeEffect)) {
      dep.add(activeEffect);
      (activeEffect as any).deps.push(dep);
     }
    }

   function triggerEffect(effect, DebuggerEventExtraInfo){
     // effect: ReactiveEffect
      if (effect.scheduler) {
         effect.scheduler()
       } else {
         effect.run()
       }
    }

    ref() ==> createRef ==> RefImpl 类初始化ref （首次逐层返回）==> 当视图调用时 ==> RefImpl > set | get ==> changed ==> 触发 triggerRefValue ==> trackEffect(重新收集依赖) ==> 调度执行

    // ReactiveEffect 类在 effect 中实现

   ```
