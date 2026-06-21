# 解除B站区域限制

Fork of [JoeyTeng/bilibili-helper](https://github.com/JoeyTeng/bilibili-helper). Use [unblock-area-limit.user.js](https://github.com/Z-H-Sun/bilibili-helper/raw/main/unblock-area-limit.user.js). For usage, see instructions in the original repository. This fork attempts to compatibilize with other BiliBili Tempermonkey scripts.

Note: This effort is mutual. The modification in the fork tries to be nice to other scripts, but any other script that hooks `window.__playinfo__` also needs to behave itself to let this script to work properly. Specifically, if another script hooks `window.__playinfo__` and makes it a property that is not `configurable`, this script will stop working. An exemplary revision to such a script is shown below. Suppose the bad script causing the conflict has the following code:

```js
let internalPlayInfo = unsafeWindow.__playinfo__; // the initial value of `window.__playinfo__`; will update in the setter
Object.defineProperty(unsafeWindow, '__playinfo__', {
    get: () => internalPlayInfo,
    set: v => {
        playInfoTransformer(v);
        internalPlayInfo = v;
    }
});
```

Since JavaScript makes a property not `configurable` by default, the code above will block this script to redefine the `__playinfo__` property, raising a `TypeError: Cannot redefine property: __playinfo__`. So, modify the code above to something like below:

```js
let internalPlayInfo = unsafeWindow.__playinfo__; // the initial value of `window.__playinfo__`; will update in the setter
const existingDescriptor = Object.getOwnPropertyDescriptor(window, '__playinfo__');
const originalGet = existingDescriptor ? existingDescriptor.get : null;
const originalSet = existingDescriptor ? existingDescriptor.set : null;
Object.defineProperty(unsafeWindow, '__playinfo__', {
    configurable: true, // necessary!
    enumerable: true,
    get: () => {
        if (originalGet) {
            let originalPlayInfo = originalGet(); // let the existing getter to run before the transform of this script runs
            playInfoTransformer(originalPlayInfo);
            return originalPlayInfo;
        }
        return internalPlayinfo; // if there is no existing getter, use the value stored in the setter
    }
    set: v => {
        if (originalSet) { originalSet(v); } // let the existing setter to run
        if (originalGet) { return; } // if this is the case, the transform is done in the getter
        playInfoTransformer(v);
        internalPlayInfo = v;
    }
});
```
