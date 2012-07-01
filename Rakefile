require 'rubygems'
require 'rake'
require 'pp'
require 'yaml'


# Configure
$config = YAML.load(File.open('config.yml').read)
pp $config

Twitter.configure do |config|
  tc = $config['twitter']
  ['consumer_key', 'consumer_secret', 'oauth_token', 'oauth_token_secret'].each do |field|
    raise "Error, config['twitter']['#{field}'] key missing. Please edit config.yml" if tc[field].nil? || tc[field.empty?
  end

  config.consumer_key = tc['consumer_key']
  config.consumer_secret = tc['consumer_secret']
  config.oauth_token = tc['oauth_token']
  config.oauth_token_secret = tc['oauth_token_secret']
end


# Helpers
def handle_tweet(status)
  tweet = status.to_openstruct
  if tweet.text.blank?
    STDERR.puts "Blank tweet text, skipping"
    STDERR.flush
    return
  end

  STDOUT.puts "Tweetscan: received \"#{tweet.user.screen_name}: #{tweet.text}\"..."
  STDOUT.flush

  if true # it's new
    puts "sending twitter msg..."
    # Post it back to Twitter
  end

  tweet
end


# void main()
task :tweetscan do

  require 'twitter/json_stream'
  $stdout.puts "Tweetscan launching..."; $stdout.flush

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

