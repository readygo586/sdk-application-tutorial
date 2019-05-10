# Keeper

一个Cosmos SDK模块的主要核心是名为`Keeper`的部分。它和存储进行交互，和其他模块的keeper进行跨模块的交互，并包含本模块的大部分核心功能。


## Keeper结构

请在`./x/nameservice/keeper.go`文件中定义`nameservice.Keeper`：

```go
package nameservice

import (
	"github.com/cosmos/cosmos-sdk/codec"
	"github.com/cosmos/cosmos-sdk/x/bank"

	sdk "github.com/cosmos/cosmos-sdk/types"
)

// Keeper maintains the link to data storage and exposes getter/setter methods for the various parts of the state machine
type Keeper struct {
	coinKeeper bank.Keeper

	storeKey  sdk.StoreKey // Unexposed key to access store from sdk.Context

	cdc *codec.Codec // The wire codec for binary encoding/decoding.
}
```



关于上述代码的几点说明：

- 3个不同的`cosmos-sdk`包被引入：
  - [`codec`](https://godoc.org/github.com/cosmos/cosmos-sdk/codec) - 提供负责Cosmos编码格式的工具——[Amino](https://github.com/tendermint/go-amino)。
  - [`bank`](https://godoc.org/github.com/cosmos/cosmos-sdk/x/bank) - `bank`模块控制账户和转账。
  - [`types`](https://godoc.org/github.com/cosmos/cosmos-sdk/types) - `types`包含了整个SDK常用的类型。
- `Keeper`结构体。在 keeper 中有几个关键部分：
  - [`bank.Keeper`](https://godoc.org/github.com/cosmos/cosmos-sdk/x/bank#Keeper) : 对`bank`模块的`Keeper`的引用。将其包括在内，可以让本模块调用`bank`模块中的函数。SDK使用[`对象能力`](https://en.wikipedia.org/wiki/Object-capability_model)来访问应用程序状态的各个部分。这是为了允许开发人员采用小权限准入原则，限制错误或恶意模块的去影响其不需要访问的状态的能力。
  - [`*codec.Codec`](https://godoc.org/github.com/cosmos/cosmos-sdk/codec#Codec) : 这是指向Amino codec的指针，该codec对binary进行编码和解码。
  - [`sdk.StoreKey`](https://godoc.org/github.com/cosmos/cosmos-sdk/types#StoreKey) : 这是将应用程序状态存持久化保存的入口(注：该入口会调用IAVL tree和数据库对应用程序状态进行存储)`sdk.KVStore`，在本应用程序中存储的是Whois struct的map,即保存的是map[name]Whois。 


## Getter 和 Setter

现在要添加`Keeper`与存储交互的方法了。首先，添加一个函数来设定name的Whois：

```go
// Sets the entire Whois metadata struct for a name
func (k Keeper) SetWhois(ctx sdk.Context, name string, whois Whois) {
	if whois.Owner.Empty() {
		return
	}
	store := ctx.KVStore(k.storeKey)
	store.Set([]byte(name), k.cdc.MustMarshalBinaryBare(whois))
}
```

在此方法中，首先使用`Keeper`中的`storeKey`获取`map[name]value`的存储对象（注：在应用程序中，所有的模块都使用同一个数据库，storeKey 可以理解为数据库中的表名， 此函数的就是在数据库k.storeKey者个表里面添加map[name]Whois这样一个结构)。

> 注意：这个函数使用[`sdk.Context`](https://godoc.org/github.com/cosmos/cosmos-sdk/types#Context)这个对象。sdk.Context包含访问像`blockHeight`和`chainID`这样信息的函数。

接下来，你可以使用KVStore的方法`.Set([]byte,[]byte)`向存储中插入`<name, value>`键值对。由于存储只接受`[]byte`,需要先使用Cosmos SDK encoding library(即 Amino)把Whois 序列化成[]byte之后再传给`Set`方法。

因为我们假定所有存在的域名的owner 必须不为空，所以如果Whois中owner栏位为空，将不会向存储中写任何东西。

接下来，添加一个函数来解析域名（即获取域名对应的Whois）：

```go
// Gets the entire Whois metadata struct for a name
func (k Keeper) GetWhois(ctx sdk.Context, name string) Whois {
	store := ctx.KVStore(k.storeKey)
	if !store.Has([]byte(name)) {
		return NewWhois()
	}
	bz := store.Get([]byte(name))
	var whois Whois
	k.cdc.MustUnmarshalBinaryBare(bz, &whois)
	return whois
}
```

这里，与`SetName`方法一样，首先使用`StoreKey`访问存储。接下来，使用`.Get([] byte) []byte`方法而不是`Set`方法。 `.Get([] byte) []byte`函数的入参是`[]byte`, 因此先把`name`字符串转化成`[]byte`传入，`.Get([] byte) []byte`函数的返回值是`[]byte`，此时我们需要需要再次用到Amino将byte slice 转换为Whois 的数据结构之后再返回。

如果是一个尚未在存储中新域名，则返回一个包含最低价格MinPrice的新Whois结构。

现在，我们需要添加根据域名获取Whois中特定参数的功能。相比于重写 store 的 getter 和 setter，我们重用了 GetWhois 和 SetWhois 函数(注：对于读操作，先通过GetWhois 获取完整的Whois 结构体，然后在获取其中Owner,Value,Price中的一个参数并返回；如果是写，则在修改Owner,Value,Price之后再调用SetWhois写入存储)。 例如，要设置某个字段，首先我们获取整个 Whois 数据，更新我们的特定字段，然后将新版本的Whois写回store。

```go
// ResolveName - returns the string that the name resolves to
func (k Keeper) ResolveName(ctx sdk.Context, name string) string {
	return k.GetWhois(ctx, name).Value
}

// SetName - sets the value string that a name resolves to
func (k Keeper) SetName(ctx sdk.Context, name string, value string) {
	whois := k.GetWhois(ctx, name)
	whois.Value = value
	k.SetWhois(ctx, name, whois)
}

// HasOwner - returns whether or not the name already has an owner
func (k Keeper) HasOwner(ctx sdk.Context, name string) bool {
	return !k.GetWhois(ctx, name).Owner.Empty()
}

// GetOwner - get the current owner of a name
func (k Keeper) GetOwner(ctx sdk.Context, name string) sdk.AccAddress {
	return k.GetWhois(ctx, name).Owner
}

// SetOwner - sets the current owner of a name
func (k Keeper) SetOwner(ctx sdk.Context, name string, owner sdk.AccAddress) {
	whois := k.GetWhois(ctx, name)
	whois.Owner = owner
	k.SetWhois(ctx, name, whois)
}

// GetPrice - gets the current price of a name.  If price doesn't exist yet, set to 1nametoken.
func (k Keeper) GetPrice(ctx sdk.Context, name string) sdk.Coins {
	return k.GetWhois(ctx, name).Price
}

// SetPrice - sets the current price of a name
func (k Keeper) SetPrice(ctx sdk.Context, name string, price sdk.Coins) {
	whois := k.GetWhois(ctx, name)
	whois.Price = price
	k.SetWhois(ctx, name, whois)
}
```

SDK 还有一个特性叫 `sdk.Iterator`，可以返回一个迭代器用于遍历指定 store 中的所有  `<Key, Value>` 对。

我们增加一个函数用于获取迭代器用于遍历 store中所有已经存在的域名。

```go
// Get an iterator over all names in which the keys are the names and the values are the whois
func (k Keeper) GetNamesIterator(ctx sdk.Context) sdk.Iterator {
	store := ctx.KVStore(k.storeKey)
	return sdk.KVStorePrefixIterator(store, []byte{})
}
```

最后需要在`./x/nameservice/keeper.go`文件中加上`Keeper`的构造函数：

```go
// NewKeeper creates new instances of the nameservice Keeper
func NewKeeper(coinKeeper bank.Keeper, storeKey sdk.StoreKey, cdc *codec.Codec) Keeper {
	return Keeper{
		coinKeeper: coinKeeper,
		storeKey:   storeKey,
		cdc:        cdc,
	}
}
```

###  接下来，会描述用户如何通过  [`Msgs` and `Handlers `](./05-msgs-handlers.md) 与刚刚建立的 store 交互。
