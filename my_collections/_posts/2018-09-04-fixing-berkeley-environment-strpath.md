---
layout: post
title:  "Fixing BerkeleyEnvironment's strPath"
---

I was [manually testing](https://github.com/bitcoin/bitcoin/pull/12493#issuecomment-417147646) the thread safety of 
reloading a Berkeley environment and I found that the three databases I had set up were not in the same 
Berkeley environment despite all being in the same directory. 

With LLDB I discovered why, the `strPath` member variable had a trailing separator in one case but not in the other:

<script src="https://gist.github.com/PierreRochard/47ae74e5c0bc618a3b1a3f5ae5443196.js"></script>

This happened because the `walletdir` in my bitcoin.conf had a trailing separator in it, but if you call `loadwallet` or 
don't specify `walletdir` the `strPath` will not have a trailing separator. Having two Berkeley database environments open 
in the same directory can cause data corruption.

I found plausible [solutions on Stack Overflow](https://stackoverflow.com/questions/36941934/parent-path-with-or-without-trailing-separator/) and the question then 
was where to place the modification. The `strPath` private member is assigned by `BerkeleyEnvironment`'s initializer 
list. 

![](/assets/berkeley_environment_strpath.png)


My first thought was to move `strPath` assignment into the `BerkeleyEnvironment` constructor body, so as to remove the 
trailing separator there. However, this approach was flawed because `BerkeleyEnvironment` objects are only created by the 
`GetWalletEnv` function if the `env_directory` in that function does not already exist as a key in the `g_dbenvs` map. 
This means that if the trailing separator removal is in the class constructor, the key in the `g_dbenvs` global object 
could still be wrong and we could still have two environments in the same directory. 

![](/assets/get_wallet_environment_calls.png)

I iterated on the solution in a [xeus-cling Jupyter notebook](https://github.com/QuantStack/xeus-cling):

```c++
#include <iostream>
#include <string>
```


```c++
#pragma cling load("libboost_filesystem")
#include <boost/filesystem.hpp>
```


```c++
namespace fs = boost::filesystem;
using namespace std;
```


```c++
class BerkeleyEnvironment
{
private:
    std::string strPath;
public:
    BerkeleyEnvironment(const fs::path& dir_path) : strPath(dir_path.string()) {};
    fs::path Directory() const { return strPath; }
}
```


```c++
std::map<std::string, BerkeleyEnvironment> g_dbenvs;
```


```c++
BerkeleyEnvironment* GetWalletEnv(const fs::path& wallet_path, std::string& database_filename)
{
    fs::path env_directory;
    if (fs::is_regular_file(wallet_path)) {
        env_directory = wallet_path.parent_path();
        database_filename = wallet_path.filename().string();
    } else {
        env_directory = wallet_path;
        database_filename = "wallet.dat";
    }
    // The fix 👇
    while ((env_directory.string().back() == '/') || (env_directory.string().back() == '\\'))
        env_directory = env_directory.remove_trailing_separator();
    return &g_dbenvs.emplace(std::piecewise_construct, std::forward_as_tuple(env_directory.string()), std::forward_as_tuple(env_directory)).first->second;
}
```


```c++
std::vector<fs::path> wallet_test_paths;
fs::path expected_path = "test/path";
std::string trailing_separators;
```


```c++
for (int i = 0; i < 4; ++i ) {
    trailing_separators += '/';
    wallet_test_paths.push_back(expected_path / trailing_separators);
}
```


```c++
for (auto& wallet_test_path: wallet_test_paths)
{
    std::string database_filename;
    BerkeleyEnvironment* env = GetWalletEnv(wallet_test_path, database_filename);
    cout << env->Directory().string() << " ← " << wallet_test_path << "\n";
}
```

test/path ← "test/path/"

test/path ← "test/path//"

test/path ← "test/path///"

test/path ← "test/path////"

```c++
g_dbenvs
```

{ "test/path" => @0x7f996d673708 }



Satisfied with this solution, I implemented it in the Bitcoin codebase, added a Boost unit test, and [submitted my pull 
request for review](https://github.com/bitcoin/bitcoin/pull/14146).

## Code Review

In their comments [ken2812221](https://github.com/bitcoin/bitcoin/pull/14146#discussion_r215100743) and 
[promag](https://github.com/bitcoin/bitcoin/pull/14146#discussion_r215099539) pointed out that the boost filesystem 
library has `fs::detail::is_directory_separator` and `fs::path::preferred_separator` respectively, which I can use to 
avoid hard coding the separator characters and, as ken2812221 points out, avoid "an infinity loop on Unix if the wallet 
filename is `wallet\`". I dropped these replacements in to my Jupyter notebook and the behavior was the same. I also 
took a look at how they are implemented in the boost filesystem library [here](https://github.com/boostorg/filesystem/blob/5a93351bfdf859ee47245e0429739226767ef0d7/include/boost/filesystem/path.hpp#L830-L850)
and [here](https://github.com/boostorg/filesystem/blob/5a93351bfdf859ee47245e0429739226767ef0d7/include/boost/filesystem/path.hpp#L63-L73) 
to make sure there were no surprises or caveats. 

In [his review](https://github.com/bitcoin/bitcoin/pull/14146#pullrequestreview-152292604) promag pointed out that the 
trailing separator removal would be better in `walletutil.cpp`'s [`GetWalletDir` function](https://github.com/bitcoin/bitcoin/blob/0.17/src/wallet/walletutil.cpp#L7). 
This seemed like a much better placement than in `GetWalletEnv`. I verified that all of the `GetWalletEnv` calls 
ultimately get their path from the `GetWalletDir` function and that the `-walletdir` configuration argument is only 
retrieved for use by the `GetWalletDir` function. I [moved the logic and rewrote the unit test](https://github.com/PierreRochard/bitcoin/commit/5a28a99d2887be85d02cb9a9a062f6bde96f56a2).
It was a good lesson: don't just look at where you found the bug, trace it back to a root cause and fix that.
