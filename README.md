# pierrerochard.github.io

### Useful commands

`npm install -g diff2html-cli`

`diff2html -F /Users/pierre/src/pierrerochard.github.io/bitcoin/diffs/pr_log-wallet-name_3_1__pr_log-wallet-name_4.html -- -M pr/log-wallet-name.3.1..pr/log-wallet-name.4`


### Bitcoin branch diffs

[pr/log-wallet-name.1..pr/log-wallet-name.2](https://pierrerochard.github.io/bitcoin/diffs/pr_log-wallet-name_1__pr_log-wallet-name_2.html)

[pr/log-wallet-name.2..pr/log-wallet-name.3](https://pierrerochard.github.io/bitcoin/diffs/pr_log-wallet-name_2__pr_log-wallet-name_3.html)

[pr/log-wallet-name.3..pr/log-wallet-name.4](https://pierrerochard.github.io/bitcoin/diffs/pr_log-wallet-name_3__pr_log-wallet-name_4.html)


diff <(git diff B A) <(git diff D A0)


`diff2html -F /Users/pierre/src/pierrerochard.github.io/bitcoin/diffs/pr_log-wallet-name_3__pr_log-wallet-name_4.html -- -M <(git show 553b1a7f815ec39dc30c65f22aa2f9f6cae41c25) <(git show 6d77ba31e760c0f1806b721e34437724da199955)`