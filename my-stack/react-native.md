# React Native Deep Dive

<!-- last-reviewed: 2026-04-09 | next-review: 2026-05-09 | confidence: b -->

---

## Core Components

| RN Component | Web Equivalent | Notes |
|---|---|---|
| `View` | `div` | Layout container |
| `Text` | `p`, `span` | All text must be inside Text |
| `Image` | `img` | Requires `source` prop |
| `TextInput` | `input` | Controlled via `value` + `onChangeText` |
| `TouchableOpacity` | `button` | Pressable with opacity feedback |
| `Pressable` | `button` | More flexible than TouchableOpacity |
| `ScrollView` | `div` with overflow | Renders all children at once |
| `FlatList` | virtualised list | Only renders visible items |

---

## ScrollView vs FlatList

**The key difference: virtualization.**

`ScrollView` renders **all children immediately** — everything in memory at once.
`FlatList` renders **only visible items** — unmounts items outside the viewport.

```tsx
// ScrollView — OK for small, fixed content
<ScrollView>
    {settings.map(item => <SettingRow key={item.id} item={item} />)}
</ScrollView>

// FlatList — required for any list that can grow
<FlatList
    data={products}
    keyExtractor={item => item.id}
    renderItem={({ item }) => <ProductCard item={item} />}
    ListEmptyComponent={<EmptyState />}
    ListHeaderComponent={<SearchBar />}
    onEndReached={loadMore}         // pagination
    onEndReachedThreshold={0.5}    // trigger when 50% from bottom
/>
```

| Use | When |
|---|---|
| `ScrollView` | Settings screen, a form, ≤ 20 static items |
| `FlatList` | Products, messages, feed, any dynamic list |

If you don't know the length — always `FlatList`.

---

## StyleSheet

No CSS. Styles are JavaScript objects. Use `StyleSheet.create` for performance (styles are validated and registered once).

```tsx
const styles = StyleSheet.create({
    container: {
        flex: 1,
        backgroundColor: '#fff',
        padding: 16,
    },
    title: {
        fontSize: 20,
        fontWeight: 'bold',
        color: '#111',
    },
});

// Flexbox is the layout model — column direction by default (unlike web which is row)
```

---

## Navigation (React Navigation)

```tsx
// Stack navigator
const Stack = createNativeStackNavigator();

function AppNavigator() {
    return (
        <NavigationContainer>
            <Stack.Navigator initialRouteName="Home">
                <Stack.Screen name="Home" component={HomeScreen} />
                <Stack.Screen name="ProductDetail" component={ProductDetailScreen} />
            </Stack.Navigator>
        </NavigationContainer>
    );
}

// Navigate from a screen
navigation.navigate('ProductDetail', { productId: item.id });

// Go back
navigation.goBack();
```

---

## WatermelonDB (Offline-First)

Local SQLite database that syncs with a backend when online. All reads/writes are local — no waiting for network.

```tsx
// Define a model
class Product extends Model {
    static table = 'products';

    @field('name') name!: string;
    @field('price') price!: number;
    @field('synced') synced!: boolean;
}

// Query
const products = await database
    .get<Product>('products')
    .query(Q.where('synced', false))
    .fetch();

// Write
await database.write(async () => {
    await database.get('products').create(product => {
        product.name = 'New Product';
        product.price = 29.99;
    });
});
```

Sync strategy: client writes locally → background sync pushes to server → server pushes changes back.

---

## Platform-Specific Code

```tsx
import { Platform } from 'react-native';

// Inline
const styles = StyleSheet.create({
    container: {
        paddingTop: Platform.OS === 'ios' ? 44 : 24,
    },
});

// Platform-specific files
// ProductCard.ios.tsx   ← loaded on iOS
// ProductCard.android.tsx ← loaded on Android
```

---

## Gaps to Fill Next

- [ ] Native modules (bridging JS to native code)
- [ ] Performance profiling (Flipper, React DevTools)
- [ ] Expo vs bare React Native tradeoffs
- [ ] Animations (Animated API, Reanimated)
- [ ] Deep linking

---

## Connected To

- `my-stack/react.md` — same mental model for components, hooks, state
- `senior/ci-cd.md` — mobile CI needs EAS Build or Fastlane, different from web
