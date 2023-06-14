Repro readme for https://github.com/cargo-lambda/cargo-lambda/issues/467

Created by following the getting started guide: https://www.cargo-lambda.info/guide/getting-started.html

1. `cargo lambda new lambda_test`

Answers: 1. Not HTTP, 2. Event type is `s3::S3Event`. (Answers aren't important.)

2. `cargo lambda build --release --arm64`

Arm64 shouldn't matter either, but it's what I used.

3. `cargo lambda deploy` 

This works! (Good!) If I log onto AWS web console, I can run test data, and it runs successfully.

Let's clean up, and then see where it breaks:

1. `rm -rf target/lambda/` to cleanup.

2. `export CARGO_TARGET_DIR=~/.cache/cargo/target` to set a shared target dir.

The exact directory doesn't matter, so long as it's not the default `./target`

3. `cargo lambda build --release --arm64` runs again - great!

4. `cargo lambda deploy` fails with:

```
Error:
  × binary file for lambda_test not found, use `cargo lambda build` to create it
```

First bug: `lambda deploy` doesn't know to use CARGO_TARGET_DIR.

Let's try to work around that:

5. `cargo lambda deploy --binary-path ~/.cache/cargo/target/lambda/lambda_test/bootstrap` Ok, this command worked! 
   Albeit with the following issue:

Usability issue: it's not obvious that you're forced to supply the name of the function. It'd be nice if it were 
required if 
--binary-path is set. To deploy to the same function name, I have to do: `cargo lambda deploy --binary-path ~/.
cache/cargo/target/lambda/lambda_test/bootstrap lambda_test`

Anyway, after running the modified command, it deployed to the correct function name. Let's run the function. Log 
into AWS console. Run test data. It fails with the error in the original ticket:

```
START RequestId: a482048b-2a77-46df-bb56-ab26cb1a6ecd Version: $LATEST
RequestId: a482048b-2a77-46df-bb56-ab26cb1a6ecd Error: Couldn't find valid bootstrap(s): [/var/task/bootstrap /opt/bootstrap]
Runtime.InvalidEntrypoint
END RequestId: a482048b-2a77-46df-bb56-ab26cb1a6ecd
REPORT RequestId: a482048b-2a77-46df-bb56-ab26cb1a6ecd	Duration: 14.25 ms	Billed Duration: 15 ms	Memory Size: 128 MB	Max Memory Used: 2 MB	
```

After doing some digging, I *believe* it's because AWS assumes it's a zip file, and not a statically linked binary. 
If there's a setting I'm missing that tells AWS to treat it as a binary, please let me know. The two options appear 
to be zip or container.

Anyway, it's configured as a zip, so let's try zipping before we pass it to deploy.

6. `cd ~/.cache/cargo/target/lambda/lambda_test/`

7. `zip -r lambda_test.zip bootstrap`

8. `cd -` to go back to the project. Let's try deploying again:

9. `cargo lambda deploy --binary-path ~/.cache/cargo/target/lambda/lambda_test/lambda_test.zip lambda_test`

Oops. This fails with:

```
Error:
  × Could not read file magic
```

Unfortunately, `lambda deploy` does not accept a architecture flag, which would allow it to then skip the "file 
magic" check.

#### Recommended solution

1. `lambda deploy` should use the CARGO_TARGET_DIR directory when looking for binaries.
2. `--binary-path` should zip the binary before uploading.
3. If `--binary-path` is used, the `NAME` should be required. The value currently defaults to the binary name 
   (`bootstrap`), and it seems unlikely someone would ever want that value to be the default.
