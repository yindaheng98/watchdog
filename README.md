# `WatchDog`

A `Door` is a goroutine, and `WatchDog` is a watchdog that can `Repair` the `Door` when it was broken.

## `UnblockedDoor`

`UnblockedDoor` can be repaired. Its `Lock` is not blocked.

```go
// House is your house
type House[P Param] interface {
    // NewDoor buy a new door for your House
    // the market maybe closed, if so, just return an error
    NewDoor() (UnblockedDoor[P], error)
}

// UnblockedDoor is the door of your house
// All the methods will SINGLE-THREADED access
type UnblockedDoor[P Param] interface {
	// Lock your Door when leaving your house
	// but some time, your Door can be Broken by some badGay
	// so you need a watchdog
	// Or maybe you can not Lock your Door, so you should buy a new door
	Lock(param P, OnBroken func(badGay error)) error

	// Repair your Door after it was Broken
	// Maybe badGay is so bad that your door can not be repair
	// you can return false and your watchdog can remove this door and buy a new door for you
	Repair(param P) error

	// Update your Door even if it was not Broken while watching
	Update(param P) error

	// Remove your Door after it was Broken
	Remove()
}
```

## `BlockedDoor`

`UnblockedDoor` cannot be repaired. Its `Lock` is blocked until return.

```go
// BlockedHouse is your house
type BlockedHouse[P Param] interface {
    // NewDoor buy a new door for your House
    // the market maybe closed, if so, just return an error
    NewDoor() (BlockedDoor[P], error)
}

// BlockedDoor is the door of your house, whose Lock func is a blocked function
// So the methods will MULTI-THREADED access
type BlockedDoor[P Param] interface {
	// BLock Lock your Door and block until some error occurred
	// 错！→×××如果出错了会直接再次调用BLock，而不是新建BlockedDoor×××
	// 多次调用了BLock，Update如何与之同步？×应该BLock只调用一次
	// 所以每个BlockedDoor中的BLock会且仅会调用一次
	// 所以请放心地把临时变量放在BlockedDoor中
	BLock(init P) error

	// Update same as UnblockedDoor.Update
	// will retry until success if return error
	Update(param P) error

	// Remove same as UnblockedDoor.Remove
	// !!! when Remove called, BLock should exit !!!
	Remove()
}
```

## `WatchDog`

```go
// WatchDog is your watchdog
type WatchDog[P Param] interface {

	// Watch let your dog start to watch your House
	Watch(init P)

	// Leave let your dog stop from watching your House
	Leave()

	Update(param P)
}
```

### `Watch`

`Watch` means you `Lock` the `Door` and do not want some `error` to open it.

So when you call `Watch`, your `WatchDog` will `Lock`(or `BLock`) the `Door` for you and `Repair`(for `Lock`) or by a `NewDoor` (for `BLock`) when some `badGay` open the door. 

### `Update`

Call the `Update` of your `Door`.

### `Leave`

Call the `Remove` of your `Door` and exit.