# Mocking in C++

These are my notes on stubs/fakes/mocks and how to make unit testing easier in C++.
I only have experience in using GTest in C++ and that's what I will be using here.

## Problem

Let's say that you need to test the following piece of functionality:

```cpp
bool IsDeviceEnabled(int device_id) {
  return ::CheckDeviceStatus(device_id) == 0;
}
```

Our `IsDeviceEnabled` method depends on some external API (`CheckDeviceStatus`)
which we don't control. This makes testing harder.

Imagine the test:

```cpp
TEST(IsDeviceEnabled, Enabled_When_Status_Zero) {
    ASSERT_TRUE(IsDeviceEnabled(1337));
}
```

While it might *work on my machine*, it probably won't provide us a deterministic
result. On some machhines the device with id 1337 won't be present.
In other environments, the device will behave diffrently and return a different status.
Immediate intuition is to mock that call. But at least with GMock, you cannot 
mock free functions. It requires some modifications to your code.

## Solution 1 - Templates
We could ask the compiler to provide us two versions of `IsDeviceEnabled`.
One using the real call to `IsDeviceEnabled`and another one that will
use some sort of fake. We can achieve that with a template:

```cpp
template<int (*CheckDeviceStatusFn)(int)>
bool IsDeviceEnabled(int device_id) {
  // The compiler will subsitute this with a 
  // function call of the given type
  return CheckDeviceStatusFn(device_id) == 0;
}

bool IsDeviceEnabled(int device_id) {
  return IsDeviceEnabled<CheckDeviceStatus>(device_id);
}
```

And the test:

```cpp
int FakeIsDeviceEnabled(int) {
    return 0;
}

TEST(IsDeviceEnabled, Enabled_When_Status_Zero) {
    ASSERT_TRUE(IsDeviceEnabled<FakeIsDeviceEnabled>(1337));
}
```

I tried using `std::function<int(int)>` but this didn't make the compiler happy.

## Solution 2

Another solution is to create an object that wraps around our library method
and take the advantage of polymophysim.
First let's define an interface:

```cpp
struct IMyApiWrapper {
    virtual ~IMyApiWrapper() = default;
    virtual int CheckDeviceStatus(int device_id) = 0;
};
```

and implement a "real" wrapper that will call the actual method:

```cpp
struct MyApiWrapper : IMyApiWrapper {
    ~MyApiWrapper() override = default;
    int CheckDeviceStatus(int device_id) override {
        return ::CheckDeviceStatus(device_id);
    }
};
```

We have to also modify `IsDeviceEnabled`:

```cpp
IMyApiWrapper* CreateWrapper() {
    return new MyApiWrapper();
}

bool IsDeviceEnabled(int device_id) {
  auto wrapper = std::make_unique<IMyApiWrapper>(CreateWrapper());
  return wrapper->CheckDeviceStatus(device_id) == 0;
}
```

And in our test we can now provide a different implementation of the wrapper
that will be our fake:

```cpp
struct FakeApiWrapper : IMyApiWrapper {
    ~FakeApiWrapper() override = default;
    int CheckDeviceStatus(int device_id) override {
        return 0;
    }
};

IMyApiWrapper* CreateWrapper() {
    return new FakeApiWrapper();
}

TEST(IsDeviceEnabled, Enabled_When_Status_Zero) {
    ASSERT_TRUE(IsDeviceEnabled(1337));
}
```

### Solution 2.1

The solution above works, assuming that the `CreateWrapper` factory method is not linked into our test binary, so that we don't get a duplicate symbol defind error.
We could further refactor this:

```cpp
using FactoryFn = std::function<IMyApiWrapper*()>;
FactoryFn g_factory_fn;

bool IsDeviceEnabled(int device_id) {
  std::unique_ptr<IMyApiWrapper> wrapper;
  if (g_factory_fn) {
      wrapper.reset(g_factory_fn());
  } else {
      wrapper = std::make_unique<MyApiWrapper>();
  }
  return wrapper->CheckDeviceStatus(device_id) == 0;
}

void SetFactoryForTesting(FactoryFn factory_fn) {
    g_factory_fn = factory_fn;
}

```cpp
struct FakeApiWrapper : IMyApiWrapper {
    ~FakeApiWrapper() override = default;
    int CheckDeviceStatus(int device_id) override {
        return 0;
    }
};

TEST(IsDeviceEnabled, Enabled_When_Status_Zero) {
    SetFactoryForTesting([]() -> IMyApiWrapper* {
        return new FakeApiWrapper();
    });
    ASSERT_TRUE(IsDeviceEnabled(1337));
}
```

### Solution 2.2

We can now easily use our mocking framework. Let's modify our test a bit:

```cpp
struct IFactory {
    virtual IMyApiWrapper* Create() = 0;
};

struct MockFactory : IFactory {
    MOCK_METHOD0(IMyApiWrapper*, Create);
};

struct MockApiWrapper : IMyApiWrapper {
    ~FakeApiWrapper() override = default;
    MOCK_METHOD1(int, CheckDeviceStatus, int);
};

TEST(IsDeviceEnabled, Enabled_When_Status_Zero) {
    testing::StrictMock<MockFactory> factory;
    testing::StrictMock<MockApiWrapper> mock_wrapper;
    SetFactoryForTesting(std::bind(IFactory::Create, factory));
    EXPECT_CALL(*mock_wrapper, CheckDeviceStatus(_)).WillOnce().Return(0);
    EXPECT_CALL(*factory, Create()).WillOnce().Return(mock_wrapper);
    ASSERT_TRUE(IsDeviceEnabled(1337));
}

```