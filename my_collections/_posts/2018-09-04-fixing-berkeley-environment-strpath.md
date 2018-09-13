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

## Code Review Round 1

In their comments [ken2812221](https://github.com/bitcoin/bitcoin/pull/14146#discussion_r215100743) and 
[promag](https://github.com/bitcoin/bitcoin/pull/14146#discussion_r215099539) pointed out that the boost filesystem 
library has `fs::detail::is_directory_separator` and `fs::path::preferred_separator` respectively, which I can use to 
avoid hard coding the separator characters and, as ken2812221 points out, avoid "an infinity loop on Unix if the wallet 
filename is `wallet\`". I took a look at how they are implemented in the boost filesystem library [here](https://github.com/boostorg/filesystem/blob/5a93351bfdf859ee47245e0429739226767ef0d7/include/boost/filesystem/path.hpp#L830-L850)
and [here](https://github.com/boostorg/filesystem/blob/5a93351bfdf859ee47245e0429739226767ef0d7/include/boost/filesystem/path.hpp#L63-L73) 
to make sure there were no surprises or caveats. 

In [his review](https://github.com/bitcoin/bitcoin/pull/14146#pullrequestreview-152292604) promag pointed out that the 
trailing separator removal would be better in `walletutil.cpp`'s [`GetWalletDir` function](https://github.com/bitcoin/bitcoin/blob/0.17/src/wallet/walletutil.cpp#L7). 
This seemed like a much better placement than in `GetWalletEnv`. I verified that all of the `GetWalletEnv` calls 
ultimately get their path from the `GetWalletDir` function and that the `-walletdir` configuration argument is only 
retrieved for use by the `GetWalletDir` function. I [moved the logic and rewrote the unit test](https://github.com/PierreRochard/bitcoin/commit/5a28a99d2887be85d02cb9a9a062f6bde96f56a2).
It was a good lesson: don't just look at where you found the bug, trace it back to a root cause and fix that.

I made these changes in my Jupyter notebook and the behavior was the same:

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
std::string walletdir_arg = "-walletdir";
```


```c++
class ArgsManager
{
protected:
    std::map<std::string, std::string> m_override_args;
public:
    void ForceSetArg(std::string& strArg, const std::string& strValue)
    {
        m_override_args[strArg] = strValue;
    }
    bool IsArgSet(std::string& strArg)
    {
        std::map<std::string, std::string>::const_iterator it = m_override_args.find(strArg);
        return it!=m_override_args.end();
    }
    std::string GetArg(std::string& strArg)
    {
        return m_override_args[strArg];
    }
}
```


```c++
ArgsManager gArgs;
```


```c++
fs::path GetWalletDir()
{
    fs::path path;
    if (gArgs.IsArgSet(walletdir_arg)) {
        path = gArgs.GetArg(walletdir_arg);
        if (!fs::is_directory(path)) {
            // If the path specified doesn't exist, we return the deliberately
            // invalid empty string.
            path = "";
        }
    }
    // Ensure that the directory does not end with a trailing separator to avoid
    // creating two Berkeley environments in the same directory
    while (fs::detail::is_directory_separator(path.string().back())) {
        path.remove_trailing_separator();
    }
    return path;
}
```


```c++
fs::path expected_wallet_dir = "test/path";
if (fs::create_directories("test")) {
    // This is the first run, create wallets subdirectory too
    fs::create_directories("test/path");
}
std::string trailing_separators;
```


```c++
for (int i = 0; i < 4; ++i) {
    trailing_separators += fs::path::preferred_separator;
    std::string database_filename;
    std::string wallet_dir_trailing = (expected_wallet_dir / trailing_separators).string();
    gArgs.ForceSetArg(walletdir_arg, wallet_dir_trailing);
    cout << GetWalletDir().string() << " ← " << wallet_dir_trailing << "\n";
}
```

test/path ← test/path/

test/path ← test/path//

test/path ← test/path///

test/path ← test/path////


## Code Review Round 2

In his second round of comments [promag](https://github.com/bitcoin/bitcoin/pull/14146#discussion_r215649285) pointed 
out that there is a `fs::canonical` function that clean up the path beyond just trailing separators. After some testing 
I agreed it was a better solution. I also searched the codebase again to identify a the best spot to insert the 
modification and found the `WalletInit::Verify` function. I realized that if I modified the `-walletdir` argument there 
then it wouldn't need to be cleaned up with every call to `GetWalletDir`. I also [added unit tests](https://github.com/bitcoin/bitcoin/pull/14146/commits/ea3009ee942188750480ca6cc273b2b91cf77ded) to make sure that 
existing `Verify` behavior was preserved.


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
std::string arg_key = "-walletdir";
```


```c++
class ArgsManager
{
protected:
    std::map<std::string, std::string> m_override_args;
public:
    void ForceSetArg(std::string& strArg, const std::string& strValue)
    {
        m_override_args[strArg] = strValue;
    }
    bool IsArgSet(std::string& strArg)
    {
        std::map<std::string, std::string>::const_iterator it = m_override_args.find(strArg);
        return it!=m_override_args.end();
    }
    std::string GetArg(std::string& strArg)
    {
        return m_override_args[strArg];
    }
}
```


```c++
ArgsManager gArgs;
```


```c++
bool Verify()
{
    if (gArgs.IsArgSet(arg_key)) {
        fs::path wallet_dir = gArgs.GetArg(arg_key);
        boost::system::error_code error;
        // The canonical path cleans the path, preventing >1 Berkeley environment instances for the same directory
        fs::path canonical_wallet_dir = fs::canonical(wallet_dir, error);
        if (error || !fs::exists(wallet_dir)) {
            return false;
        } else if (!fs::is_directory(wallet_dir)) {
            return false;
        // The canonical path transforms relative paths into absolute ones, so we check the non-canonical version
        } else if (!wallet_dir.is_absolute()) {
            return false;
        }
        gArgs.ForceSetArg(arg_key, canonical_wallet_dir.string());
    }
    return true;
}
```


```c++
struct TestCase
{
    std::string label;
    fs::path input_path;
    bool expected_result;
}
```


```c++
std::vector<TestCase> test_cases;

std::string sep;
sep += fs::path::preferred_separator;

fs::create_directories("tempdir");
fs::path m_cwd = fs::current_path();
fs::path m_datadir = m_cwd / "tempdir";
fs::current_path(m_datadir);

test_cases.push_back(TestCase {"default", m_datadir / "wallets", true});
test_cases.push_back(TestCase {"custom", m_datadir / "my_wallets", true});
test_cases.push_back(TestCase {"trailing", m_datadir / "wallets" / sep, true});
test_cases.push_back(TestCase {"trailing2", m_datadir / "wallets" / sep / sep, true});

test_cases.push_back(TestCase {"nonexistent", m_datadir / "path_does_not_exist", false});
test_cases.push_back(TestCase {"file", m_datadir / "not_a_directory.dat", false});
test_cases.push_back(TestCase {"relative", "wallets", false});

fs::create_directories(m_datadir / "wallets");
fs::create_directories(m_datadir / "my_wallets");
std::ofstream f((m_datadir / "not_a_directory.dat").BOOST_FILESYSTEM_C_STR);
f.close();
```


```c++
void SetWalletDir(const fs::path& walletdir_path)
{
    gArgs.ForceSetArg(arg_key, walletdir_path.string());
}
```


```c++
for (auto test_case: test_cases)
{
    std::cout << test_case.label << std::endl;
    SetWalletDir(test_case.input_path);
    bool result = Verify();
    if (result) {
        fs::path walletdir = gArgs.GetArg(arg_key);
        fs::path expected_path = fs::canonical(test_case.input_path);
        fs::path wallet_dir = gArgs.GetArg(arg_key);
        cout << wallet_dir.string() << " ← " << test_case.input_path.string() << std::endl << std::endl;
    }
}
```

default

/Users/pierre/src/cling-notebooks/bitcoin/tempdir/wallets ← /Users/pierre/src/cling-notebooks/bitcoin/tempdir/wallets


custom

/Users/pierre/src/cling-notebooks/bitcoin/tempdir/my_wallets ← /Users/pierre/src/cling-notebooks/bitcoin/tempdir/my_wallets


trailing

/Users/pierre/src/cling-notebooks/bitcoin/tempdir/wallets ← /Users/pierre/src/cling-notebooks/bitcoin/tempdir/wallets/


trailing2

/Users/pierre/src/cling-notebooks/bitcoin/tempdir/wallets ← /Users/pierre/src/cling-notebooks/bitcoin/tempdir/wallets//


nonexistent

file

relative



```c++
fs::current_path(m_cwd);    
fs::remove_all(m_datadir);
```
