# Qt

//WIP

Here is how you would generate Qt signals's implementations using some of the constructs introduced above.
```

constexpr {
    for-> (auto && m : std::reflexpr->(C).methods()) {
        if(!m.has_attribute("qt:signal"))
            continue;
        auto count = sizeof...(fun.parameters());
        auto hasRet = std::is_same_v<void, typename(fun)>;
        if(!hasRet && count == 0)   ->  {
            auto fun(typename args...)   {
                QMetaObject::activate(this, &staticMetaObject, 0, nullptr);
                
            }
        }
        else if (!hasRet) -> { 
            auto fun(typename args...)    {
                void *_a[] = { nullptr,  to_opaque_ptr<>(args)... };
                QMetaObject::activate(this, &staticMetaObject, 0, _a);
            }
        }
        else -> {
            auto fun(typename args...)  {
                typename(fun) ret;
                void *_a[] = { to_opaque_ptr<>(ret), to_opaque_ptr<>(args)... };
                QMetaObject::activate(this, &staticMetaObject, 0, _a);
                return ret;
            }
        }
    }
}

```
