# Hot code-reloading

By default, apps are restarted when new versions are deployed. This is to make it easy for people to deploy apps quickly with quick setup.

### To enable

* Set the `deploy_type` variable to `upgrade` in your project's `playbooks/vars/main.yml` file
* Everytime you deploy, update the app's version in the project's `mix.exs`


### Automatic versioning using git

For hot code-reloading, the app's version needs to be updated in `mix.exs` for every deploy. That can get repeatitive, since hobby projects are updated & deployed frequently. We've got a workaround for that. We'll use automatic versioning based on git commit SHAs. Technical details are explained in [this blog post](TODO).

* Change the following in `mix.exs`

```elixir
def project do
  [app: :hello_phoenix,
   version: "1.4.1",
   elixir: "~> 1.0",
   ...
```

to look like the following

```elixir
def project do
  {result, _exit_code} = System.cmd("git", ["rev-parse", "HEAD"])

  # We'll truncate the commit SHA to 7 chars. Feel free to change
  git_sha = String.slice(result, 0, 7)

  [app: :hello_phoenix,
   version: "1.4.1-#{git_sha}",
   elixir: "~> 1.0",
   ...
```

That just changes the `version` to use the git commit SHA.


###Update

Using commit SHA will not work with distillery.

The issue with git_sha is that it’s not ordered. The next version is assumed to have a number that is greater than the previous one by trivial alphanumeric sort. With sha that’s not the case.

You might use date for that purpose. E. g.
```
git log -1 --date=raw --format=%cd
#⇒ 1535467693 +0200 # seconds since epoch
```
Now let’s elixir it:
```
{epoch, _} = System.cmd("git", ~w|log -1 --date=raw --format=%cd|)
[sec, tz] =
  epoch
  |> String.split(~r/\s+/, trim: true)
  |> Enum.map(&String.to_integer/1)
#⇒ [1527769224, 200]
sec + tz * 36 # * 60 * 60 / 100
#⇒ 1527776424
```
The number above is always growing.

Sidenote: use inplace binary pattern match instead of String.slice/3 whenever possible:
```
{<<git_sha::binary-size(8), _rest::binary>>, _exit_code} =
  System.cmd("git", ["rev-parse", "HEAD"])
#⇒ {"556c53987eb55c82ffb6925f9f56eae5de01c119\n", 0}
git_sha
#⇒ "556c5398"
```

So in your application's mix.exs file

```elixir
  defp versions do
    {epoch, _} = System.cmd("git", ~w|log -1 --date=raw --format=%cd|)
    [sec, tz] =
      epoch
      |> String.split(~r/\s+/, trim: true)
      |> Enum.map(&String.to_integer/1)
    sec + tz * 36
  end
```

and do this way

```elixir
def project do
  [app: :hello_phoenix,
   version: "1.4.1-#{versions()}",
   elixir: "~> 1.0",
   ...
```

You might do whatever you want, the date is constantly growing with time, unlike sha, so it’s applicable for your needs. Also you might use commit number, or whatever constantly growing.
