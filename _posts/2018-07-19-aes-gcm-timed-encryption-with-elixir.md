---
layout: post
title:  "AES GCM timed encryption with Elixir"
author: Pedro G. Galaviz
date:   2018-07-19 12:00:00 -0600
comments: true
tags: [elixir, security]
---

While developing elixir web apps sometimes we'll find the necessity of generating
encrypted tokens, these can be used in transactional emails (account confirmation,
password resets, etc.) or for other purposes. While there are packages out there
than can help us with this, I wanted to explore more in dept what are the bare options:

## Cryptography in Elixir

While I'm no expert in cryptography by any means, it's always a good idea to know
what our options are in this matter, fortunately in Elixir we don't have to
reinvent the wheel (which is always a bad idea when dealing with security concerns),
the **Beam** includes a very helpful module called **crypto**.

If you're not used to Erlang, it's documentation may seem a little fuzzy when
compared to Elixir's. **crypto** offers us helpful functions to work with
different hashing and encrypting functions that go from MD5, DES, AES, RSA and
others.

For the purpose of this experiment we'll use the AES (Advanced Encryption
Standard), while for simple experimenting DES could help, it's not considered
secure. So we'll go the AES in GCM (Galois Counter Mode), you can learn more
about it [here](https://en.wikipedia.org/wiki/Galois/Counter_Mode).

## The implementation

Let's get started, create a new application:

{% highlight shell %}
mix new crypto
cd crypto
{% endhighlight %}

Since we want to make tokens with an expiration time, let's add **Timex** as a
dependency to work comfortably with dates and time.

{% highlight elixir %}
# mix.exs
# ...
defp deps do
  [
    {:timex, "~> 3.3"}
  ]
end
{% endhighlight %}

Install it with `mix deps.get` and we're all set to start our
implememtation.

As we're using symmetric encryption we need a single **key** with which we can encrypt
and decrypt our messages, AES allows us to use 128, 192 or 256 bits keys, we'll
create 256 bits keys. Open your **Crypto** module and add the followinf
function:

{% highlight elixir %}
# lib/crypto.ex
# ...
def generate_key do
  :crypto.strong_rand_bytes(32) |> Base.url_encode64()
end
{% endhighlight %}

Here we're using Erlang's **crypto** module to generate a 256 bits binary and
then encoding it with base 64 so it's more user friendly.

Now we can easily generate keys by running `Crypto.generate_key()` on an `iex`
session.

We can use the returned String and add it to our environment configuration:

{% highlight elixir %}
# config/dev.exs
config :crypto,
  crypto_key: "lVPevoQt_xR5X7oMuHqTfSmLlHtTCQ4dZZJasS_cFMw="
# config/test.exs
config :crypto,
  crypto_key: "lVPevoQt_xR5X7oMuHqTfSmLlHtTCQ4dZZJasS_cFMw="
# config/prod.exs
config :crypto,
  crypto_key: System.get_env("CRYPTO_KEY")
{% endhighlight %}

### Encryption

Now we're ready to start encrypting messages. Let's create 2 functions
`encrypt/1` and `encrypt/2`, both will allow us to encrypt a `String` but one
will allow us to add an expiration time in minutes.

{% highlight elixir %}
# lib/crypto.ex
# ...
def encrypt(data) when is_binary(data) do
  # ...
end
def encrypt(_), do: {:error, :invalid}

def encrypt(data, minutes) when is_binary(data) do
  # ...
end
def encrypt(_, _), do: {:error, :invalid}
{% endhighlight %}

Now let's create a private function that will actually encrypt the message:

{% highlight elixir %}
# lib/crypto.ex
@key Application.get_env(:crypto, :crypto_key) |> Base.url_decode64!()
@auth_data "my-crypto-module"
# ...
defp block_encrypt(data) do
  iv = :crypto.strong_rand_bytes(16)
  case :crypto.block_encrypt(:aes_gcm, @key, iv, {@auth_data, data, 16}) do
    {cipher_text, cipher_tag} ->
      {:ok, {iv, cipher_text, cipher_tag}}
    x ->
      {:error, x}
  end
end
{% endhighlight %}

Here we're adding 2 module attibutes, the first one `@key` is getting the key we
previously set in the app configuration files, the second one `@auth_data` is an
arbitrary String thats known as AAD (Associated Authenticated Data) and is part
of the AES GCM specification. The AAD has nothing to do with making it "more secure".
The aim of AAD is to attach information to the ciphertext (the encrypted message)
that is not encrypted, but is bound to it in the sense that it
cannot be changed or separated.

Inside the function we're generating a variable `iv` that referst to the
**Initialization Vector** and it must be always random, according to the GCM
information, part of the strenght of our encryption depends on it. Here the `iv`
is a 128 bits random String.

Then we'll use the **crypto** function **block_encrypt/4**, the first argument
is the algorithm we're using, in this case `:aes_gcm`, next our `@key`, then the
`iv` and last a tuple which contains our `@auth_data`, the message we want to
encrypt and lastly the length of the `cipher_tag` to generate, it can be in the
range of `1..16` the bigger the better.

If all goes well, the **crypto** function will return a tuple with the
`cipher_text` and the `cipher_tag`, in our implementation we'll return:

{% highlight elixir %}
{:ok, {iv, cipher_text, cipher_tag}}
{% endhighlight %}

or an error tuple if it occurs.

Going back to our `encrypt/1` function we can now add its implementation:

{% highlight elixir %}
# lib/crypto.ex
# ...
def encrypt(data) when is_binary(data) do
  case block_encrypt(data) do
    {:ok, payload} -> {:ok, payload}
    {:error, reason} -> {:error, reason}
  end
end
# ...
{% endhighlight %}

Here if the encryption takes place we're returning `{:ok, payload}` where
payload is a 3 item tuple. We can do better, lets encode our tuple so we can
work with it:

{% highlight elixir %}
# lib/crypto.ex
# ...
defp encode_payload({iv, cipher_text, cipher_tag}) do
  Base.url_encode64(iv <> cipher_tag <> cipher_text)
end
{% endhighlight %}

The `encode_payload/1` function is first concatenating our 3 items and then
encoding them to base 64. Notice how we're changing the `cipher_text` and
`cipher_tag` order, this is important as you'll see later.

Lets update the `encrypt/1` function to return our encoded data:

{% highlight elixir %}
# lib/crypto.ex
# ...
def encrypt(data) when is_binary(data) do
  case block_encrypt(data) do
    {:ok, payload} -> {:ok, encode_payload(payload)}
    {:error, reason} -> {:error, reason}
  end
end
# ...
{% endhighlight %}

Now our function will return fully working **Tokens** that we can use in the
outside!

What about the `encrypt/2` function? Lets work on that now:

{% highlight elixir %}
# lib/crypto.ex
# ...
def encrypt(data, minutes) when is_binary(data) do
  ttl = Timex.shift(Timex.now(), minutes: minutes)
  data = data <> "|" <> to_string(ttl)
  case block_encrypt(data) do
    {:ok, payload} -> {:ok, encode_payload(payload)}
    {:error, reason} -> {:error, reason}
  end
end
# ...
{% endhighlight %}

Here we're using the **Timex** library to generate a `ttl` variable which will
be the time our token expires, then we concatenate it to our data after
transforming it to a String. (Notice we're using "|" as a separator)

And we're all set to encrypting messages and generating tokens, next:
decryption.

### Decryption

Any encryption would be pointless if we can't decrypt the messages generated,
lets create a `decrypt/1` function to do it.

{% highlight elixir %}
# lib/crypto.ex
# ...
def decrypt(token) when is_binary(token) do
  #...
end
def decrypt(_), do: {:error, :invalid}
# ...
{% endhighlight %}

Remember our **Token** is base 64 encoded, and that we concatenated the data,
lets create a function to decode it:

{% highlight elixir %}
# lib/crypto.ex
# ...
defp decode_payload(encoded_token) do
  with {:ok, decoded} <- Base.url_decode64(encoded_token) do
    case decoded do
      << iv::binary-size(16), cipher_tag::binary-size(16), cipher_text::bitstring >> ->
        {:ok, {iv, cipher_text, cipher_tag}}
      _ ->
        {:error, :invalid}
    end
  else
    :error -> {:error, :invalid}
  end
end
# ...
{% endhighlight %}

This is our first line of defense, if **Token** is not correctly base 64 encoded
we can return an error. If decoding takes place then we can pattern match
against it to split the previoulsy concatenated info. Remember we changed the
`cipher_text` and `cipher_tag` order? Well, here's why we did it: the `iv` and
the `cipher_tag` have a length of 16 while our message String has an arbitrary
and variable length, so in order to pattern match against it the order is important.

If all goes well we return `{:ok, {iv, cipher_text, cipher_tag}}`, as you can se
it reverts the order as we had it previously to base 64 encoding.

With the decoded payload we can now attempt to decrypt it, lets implement a
`block_decrypt/1` function:


{% highlight elixir %}
# lib/crypto.ex
# ...
defp block_decrypt({iv, cipher_text, cipher_tag}) do
  case :crypto.block_decrypt(:aes_gcm, @key, iv, {@auth_data, cipher_text, cipher_tag}) do
    :error -> {:error, :invalid}
    data -> {:ok, data}
  end
end
# ...
{% endhighlight %}

This function accepts a tuple which resembles our decoded payload, and then
it will use Erlang's **crypto** module to decrypt it using **block_decrypt/4**.

If decryption works we'll get a `{:ok, data}` response, where **data** is the
encrypted message.

Now lets implement our `decrypt/1` function:

{% highlight elixir %}
# lib/crypto.ex
# ...
def decrypt(token) when is_binary(token) do
  with \
    {:ok, payload} <- decode_payload(token),
    {:ok, decrypted} <- block_decrypt(payload)
  do
    case String.split(decrypted, "|") do
      [data, expiration] ->
        {:ok, date_time} = Timex.parse(expiration, "{ISO:Extended:Z}")
        if Timex.after?(date_time, Timex.now()) do
          {:ok, data}
        else
          {:error, :expired}
        end
      [data] ->
        {:ok, data}
    end
  else
    _ -> {:error, :invalid}
  end
end
# ...
{% endhighlight %}

First we attempt to decode the **Token**, then to decrypt it, if this 2 steps
went well we can now pattern match against the data after we try to split it
when the "**|**" is found.

If an **expiration** is found first we parse it so we can work with it using
**Timex**, then we compare the **expiration** to the current time. If the token
is not expired we return the data, or an error if it is.

If no **expiration** is found, we just return the data.

The complete module should look something like this:


{% highlight elixir %}
# lib/crypto.ex
defmodule Crypto do
  @moduledoc """
  Provides functions to encrypt and decrypt base_64 encoded tokens with
  AES in GCM algorithm, accept an optional timer if expiration minutes are given.
  """

  @auth_data "my-crypto-module"
  @key Application.get_env(:crypto, :crypto_key) |> Base.url_decode64!()

  @doc """
  Generate a 256 bits url 64 encoded key
  """
  def generate_key do
    :crypto.strong_rand_bytes(32) |> Base.url_encode64()
  end

  @doc """
  Encrypts a string and returns a token
  """
  @spec encrypt(String.t) :: {:ok, String.t} | {:error, any}
  def encrypt(data) when is_binary(data) do
    case block_encrypt(data) do
      {:ok, payload} ->
        {:ok, encode_payload(payload)}
      {:error, reason} ->
        {:error, reason}
    end
  end
  def encrypt(_), do: {:error, :invalid}

  @doc """
  Same as encrypt/1 but adds a given expiration time in minutes
  """
  @spec encrypt(String.t, integer) :: {:ok, String.t} | {:error, any}
  def encrypt(data, minutes) when is_binary(data) do
    ttl = Timex.shift(Timex.now(), minutes: minutes)
    data = data <> "|" <> to_string(ttl)
    case block_encrypt(data) do
      {:ok, payload} ->
        {:ok, encode_payload(payload)}
      {:error, reason} ->
        {:error, reason}
    end
  end
  def encrypt(_, _), do: {:error, :invalid}

  @doc """
  Decrypts a given token, if expiration exist evaluate it.
  """
  @spec decrypt(String.t) :: {:ok, String.t} | {:error, :invalid} | {:error, :expired}
  def decrypt(token) when is_binary(token) do
    with \
      {:ok, payload} <- decode_payload(token),
      {:ok, decrypted} <- block_decrypt(payload)
    do
      case String.split(decrypted, "|") do
        [data, expiration] ->
          {:ok, date_time} = Timex.parse(expiration, "{ISO:Extended:Z}")
          if Timex.after?(date_time, Timex.now()) do
            {:ok, data}
          else
            {:error, :expired}
          end
        [data] ->
          {:ok, data}
      end
    else
      _ -> {:error, :invalid}
    end
  end
  def decrypt(_), do: {:error, :invalid}

  # =================
  # Private functions
  # =================

  defp block_encrypt(data) do
    iv = :crypto.strong_rand_bytes(16)
    case :crypto.block_encrypt(:aes_gcm, @key, iv, {@auth_data, data, 16}) do
      {cipher_text, cipher_tag} ->
        {:ok, {iv, cipher_text, cipher_tag}}
      x ->
        {:error, x}
    end
  end

  defp block_decrypt({iv, cipher_text, cipher_tag}) do
    case :crypto.block_decrypt(:aes_gcm, @key, iv, {@auth_data, cipher_text, cipher_tag}) do
      :error -> {:error, :invalid}
      data -> {:ok, data}
    end
  end

  # Returns a base_64 encoded token
  #  init_vec  <> cipher_tag <> cipher_text
  # [128 bits] <> [128 bits] <> [??? bits]
  # Change text & tag order so we can pattern match binary later.
  defp encode_payload({iv, cipher_text, cipher_tag}) do
    Base.url_encode64(iv <> cipher_tag <> cipher_text)
  end

  # Decodes and splits a token
  defp decode_payload(encoded_token) do
    with {:ok, decoded} <- Base.url_decode64(encoded_token) do
      case decoded do
        << iv::binary-size(16), cipher_tag::binary-size(16), cipher_text::bitstring >> ->
          {:ok, {iv, cipher_text, cipher_tag}}
        _ ->
          {:error, :invalid}
      end
    else
      :error -> {:error, :invalid}
    end
  end
end
{% endhighlight %}

Now we can use our module like this:

{% highlight elixir %}
{:ok, token} = Crypto.encrypt("John Doe", 5)
{:ok, message} = Crypto.decrypt(token)
assert message == "John Doe"
{% endhighlight %}

As long as 5 minutes haven't passed, our token will be valid, after that we'll
get a `{:error, :expired}` error.

## Conclusion

This experimet shows us that Elixir has the capacity and tools for using
encryption thanks to the Erlang/Beam environment.

While this code is probably not production ready, it's been fun to implement,
specially the pattern match parts which makes the code more concise and
friendlier to read.

If you have any observation or something to add please don't hesitate and leave
a comment below.

