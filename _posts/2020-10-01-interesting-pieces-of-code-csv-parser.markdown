---
layout: post
title:  "Interesting pieces of code- CSV parser"
subtitle: "code I find interesting"
date:   2020-10-01 18:59:45
categories: [code]
---

This is an ongoing series of interesting code snippets, what they do and why I used them.

___

Here is a CSV parser.  I wrote this for an interview one time, actually, I wrote it wrong twice, and then wrote this, which is less wrong and more succinct.

I like this because it takes the essential building blocks of any CSV file, namely a row and a field, and iterates through those to `split_elements` for each of them.  The `split_elements` method serves both purposes and is a general purpose method that is likable.  Finally, when we've reached the lowest divisable level by splitting the split rows, we call the less likeable method `unquote`.  I think Ruby has provided us ham-fisted tools for what this method does, or maybe I was ham fisted when I wrote this having expended my creative energies on the `split_elements` method.  The `unquote` here has the uneviable task of matching arbitrary quote chars which are not preceeded by escape chars.  Think about something like `"foo says \"bar\""`.  As we iterate through each character of the string after the first (which we expect to be a quote char if the field contains a quote), we have three choices.

1. We're dealing with the char _after_ a quote char
2. We're dealing with an escape char, something like `\`.
3. We're dealing with some other character

At first glance, the splitting of elements shouldn't be so complicated, after all can't we just iterate through until we find an unescaped quote character?.  The answer is no, and this was why I implemented this wrongly twice before writing this.  It is wrong becuase of _nested quotes_.  We must split rows, then fields holistically, and cannot determine if we are in a row or field without looking at the document as a whole.  Since we do have to look at the whole document, this exposes a potential attack vector for this piece of code.  Think about what would happen if this code was given a massive document, with millions of rows (or millions of fields in one row).  Attempting to split and process each of the documents elements would exhaust the Ruby VM's memory.  It is for this reason that the solution here is a fun thing to submit for an interview, but is not something you'd want to use for real.

I did not take the job, but I appreciated the nature of the code challenge.

{% highlight ruby %}
class CSV
  attr_reader :col_sep, :row_sep, :quote_char

  def self.parse(string, col_sep = ",", quote_char = '"', row_sep = "\n")
    new(string, col_sep, quote_char, row_sep).parse!
  end

  def initialize(string, col_sep = ",", quote_char = '"', row_sep = "\n")
    @input      = string
    @col_sep    = col_sep
    @row_sep    = row_sep
    @quote_char = quote_char
  end

  def parse!
    split_elements(@input, row_sep).map do |row|
      split_elements(row, col_sep).map do |field|
        unquote(field)
      end
    end
  end

  private

  def unquote(field)
    return field unless field[0] == quote_char
    buffer = ""
    escape = false
    field.chars.each do |char|
      if escape
        buffer << char
        escape = false
      elsif char == quote_char
        escape = true
      else
        buffer << char
      end
    end
    buffer
  end

  def split_elements(string, separator)
    arr = string.split(separator)
    return [""] if arr.length == 0
    acc = []
    loop do
      break if arr.length == 0
      acc << arr.shift
      loop do
        break if acc.last.scan(quote_char).count.even?
        raise ArgumentError.new("unclosed quote") if arr.length == 0
        acc.last << (separator + arr.shift)
      end
    end
    acc
  end
end
{% endhighlight %}
