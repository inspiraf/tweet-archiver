require 'rubygems'
require 'bundler'
Bundler.setup

require 'system_timer'
require 'json'
require 'twitter'
require 'twitter/json_stream'
require 'imgur2'

require 'pp'
require 'fileutils'
require 'yaml'

require 'to_openstruct'


# Configure
$config = YAML.load(File.open('config.yml').read)
pp $config

Twitter.configure do |config|
  tc = $config['twitter']
  ['consumer_key', 'consumer_secret', 'oauth_token', 'oauth_token_secret'].each do |field|
    raise "Error, config['twitter']['#{field}'] key missing. Please edit config.yml" if tc[field].nil? || tc[field].empty?
  end

  config.consumer_key = tc['consumer_key']
  config.consumer_secret = tc['consumer_secret']
  config.oauth_token = tc['oauth_token']
  config.oauth_token_secret = tc['oauth_token_secret']
end


# Helpers
def handle_tweet(status)
  tweet = status.to_openstruct
  if tweet.nil?
    STDERR.puts "Bad tweet man"; STDERR.flush
    return
  elsif tweet.text.empty?
    STDERR.puts "Blank tweet text, skipping"; STDERR.flush
    return
  end

  # Build the url to the tweet
  # why isn't this in the status obj? Weird
  user_url = "http://twitter.com/#{tweet.user.screen_name}"
  tweet_url = "#{user_url}/status/#{tweet.id_str}"
  STDOUT.puts "Tweetscan: received \"#{tweet.user.screen_name}: #{tweet.text}\" => #{tweet_url}"
  STDOUT.flush

  if true # it's new
    # Save it to disk
    puts "Saving tweet to disk..."
    json_file = "tweets/#{tweet.user.screen_name}-#{tweet.id_str}.json"
    File.open(json_file, 'w') {|f| f.write(status.to_json) }

    # Post it back to Twitter (?)
    # ...

    # Take a screenshot -- just the fullscreen one pls
    # TODO fork into background
    puts "Capturing screenshot..."
    output = "screenshots/#{tweet.user.screen_name}-#{tweet.id_str}"
    `webkit2png #{tweet_url} -F -o #{output}`
    screenshot_file = "#{output}-full.png" # what webkit2png actually makes (using -F)

    # Upload said image to imgur
    if $config['imgur']
      puts "Uploading to imgur..."
      imgur = Imgur2.new($config['imgur']['api_key'])
      upload = imgur.upload(File.open(screenshot_file))
      pp upload
    end

    # Upload that shit to S3 ...
    # TODO

    puts "Done"
  end

  tweet
end


# void main()
task :tweetscan do

  $stdout.puts "Tweetscan launching..."; $stdout.flush

  # Make our directory for screenshots and saving tweets
  FileUtils.mkdir_p 'tweets'
  FileUtils.mkdir_p 'screenshots'

  # Run our query
  raise "No query key specified in config" if $config['query'].nil? || $config['query'].empty?

  search_query = $config['query']
  path = "/1/statuses/filter.json?#{search_query}"
  $stdout.puts "path=#{path.inspect}"; $stdout.flush

  EventMachine::run {
    stream = Twitter::JSONStream.connect(
      :path    => path,
      :oauth => {
        :consumer_key    => ENV['CONSUMER_KEY'] || Twitter.consumer_key,
        :consumer_secret => ENV['CONSUMER_SECRET'] || Twitter.consumer_secret,
        :access_key      => ENV['ACCESS_KEY'] || Twitter.oauth_token,
        :access_secret   => ENV['ACCESS_SECRET'] || Twitter.oauth_token_secret,
      },
      :ssl => true
    )

    stream.each_item do |item|
      json = JSON.parse(item)
      handle_tweet(json)
    end

    stream.on_error do |message|
      $stdout.print "Tweetscan error: #{message}\n"
      $stdout.flush
    end

    stream.on_reconnect do |timeout, retries|
      $stdout.print "Tweetscan reconnecting in: #{timeout} seconds\n"
      $stdout.flush
    end

    stream.on_max_reconnects do |timeout, retries|
      $stdout.print "Tweetscan failed after #{retries} failed reconnects\n"
      $stdout.flush
    end

    trap('TERM') {
      stream.stop
      EventMachine.stop if EventMachine.reactor_running?
    }
  }
end
