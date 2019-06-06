---
layout: post
title: 'From the archives: Learning to Sight Along the Space Capsule'
---

*This post was originally published on 2016-12-29 when I was on the Data Engineering team at Grand Rounds. Enjoy!*

---

Testing does not have to be complicated, but sometimes it can turn into spaghetti just like any other code. [This talk](https://www.youtube.com/watch?v=URSWYvyc42M) by Sandi Metz at Railsconf 2013 inspired me to revisit some of the unit tests for one of our services that relies heavily on FTP and SFTP connections.

##### Why are my tests failing?

The previous test suite for this service used a gem that was originally created as a mocked framework for testing FTP connection behaviors. However, a few key problems existed with this solution:

- The gem hasn’t been maintained in a couple years, which started to cause intermittent spec failures.
- The gem doesn't support SFTP connections, which our service relies much more heavily on than FTP.
- The gem doesn't actually mock anything. It starts up a local FTP server, runs the tests, and then shuts the server down. Each test that’s run against it is generating a real response, albeit locally. This not only took up a lot of time (starting and stopping the local server before and after each spec), but was also another source of intermittent failures.

So, after creating a plan of attack, I rewrote or deleted every single test that used this gem. Each test was also re-evaluated for "sighting along the space capsule," as Sandi Metz calls it. Any test that was written for a private method, a message sent to the object’s self, or an implementation detail was scrutinized (and most of them deleted), which exposed the true underlying issue: almost every single test we'd written that used the gem was an *implementation* test, not an *interface* test.

External responses are supposed to be completely mocked out in unit testing, not make actual calls to an API or server. Tests exist to make sure *our* code works as expected, not to verify that no errors are raised when making external requests. A unit test that requires a response from an outside source is already a broken test; save those for integration tests instead.

##### Reframing our point of view

Here’s an example to illustrate my point. Let’s say we have an object whose job is to get a list of files from a specific folder on an FTP server. For brevity’s sake, let’s define it like this, with the nitty gritty details fleshed out elsewhere:

```
    class FtpThing
      def get_list_of_files(username, password, folder_name)
        @some_connection.open(username, password)
        list_of_files = @some_connection.list(folder_name)
        @some_connection.close
        list_of_files
      end
    end
```

Now, one option for testing this is the way we did it before, something like this:

```
    let(:ftp_thing) { FtpThing.new }
    let(:list) { [‘file1.txt’, ‘file2.txt'] }

    before do
      # spin up the “mock" (real) server
      # put a (real) file or two in a folder on the server
    end

    it ‘gives me the right list of files’ do
      expect(ftp_thing.get_list_of_files(‘username’, ‘password’, ‘folder_name’)).to eq(list)
      ftp_thing.get_list_of_files(‘username’, ‘password’, ‘folder_name’)
    end

    after do
      # shut down the server
    end
```

On the surface, this doesn’t look terrible. It’s testing the value returned from `getListOfFiles`, right? Yay, that’s what we want! But... not quite. We’ve started a real server, and we hope that a) it started in time for our tests to run, and b) it doesn’t fail or raise any errors, which is how our tests will pass. Instead, we need to `allow` a mocked response and set up our expectation this way:

```
    let(:mock_connection) { double(‘ftp’) }
    let(:ftp_thing) { FtpThing.new }
    let(:list) { [‘file1.txt’, ‘file2.txt'] }

    before do
      ftp_thing.set_instance_variable(:@some_connection, mock_connection)
      allow(mock_connection).to receive(:open)
      allow(mock_connection).to receive(:list).and_return(list)
      allow(mock_connection).to receive(:close)
    end

    it ‘gives me the right list of files’ do
      expect(ftp_thing.get_list_of_files(‘username’, ‘password’, ‘folder_name’)).to eq(list)
      ftp_thing.get_list_of_files(‘username’, ‘password’, ‘folder_name’)
    end
```

See the difference? We aren’t starting a server or making a real external call; we’re just pretending that it happened and gave us the right response. The true test is whether *our* code is giving us the right value back after it makes an external call. This is also an easy way to test error responses, too! I can allow my connection to raise an error, and then test my error handling from there, rather than trying to fake out a whole server thing.

Bonus points: If I want to change the implementation of my server call--maybe I want to change how it connects or opens the connection, or something similar--then my tests are not broken and I can continue on my merry way.

##### The moral of the story

No project is complete without some beautiful numerical data to back it up, so here’s some numbers to show off just how much time this saves us. Running the entire test suite on the application before:

```
    Finished in 2 minutes 56.1 seconds (files took 2.72 seconds to load)
    172 examples, 0 failures
```

And after:

```
    Finished in 5.67 seconds (files took 2.77 seconds to load)
    155 examples, 0 failures
```

That’s a *97% time improvement.* Precious minutes of time regained, and no more intermittent failures as well.

The moral of the story? Don’t make real calls to real servers in your tests, even if they’re local to your machine or CI container. Set up mocked responses when needed, and test that *your* code is actually working, not theirs. (And seriously: watch that talk by Sandi Metz at least three more times until it really sticks. Your future self will thank you.)
