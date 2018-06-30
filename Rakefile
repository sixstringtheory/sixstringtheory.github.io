desc 'Install dev dependencies.'
task :init do
  sh 'brew install exiftool imagemagick'
  sh 'gem install octopress-paginate'
end

desc 'Build all SASS and Jekyll sources.'
task :build do
	_build
end

desc 'Build everything necessary to render a photo gallery from a directory of images: build Jekyll YAML front matter for album and each photo, render thumbnails, and an index for the gallery. At the end, runs everything in `rake build`.'
task :prepare_photo_gallery,[:dir] do |t, args|
  _prepare_photo_gallery args[:dir]
end

desc 'Build all the photo galleries. Helpful to backfill changes to the build process.'
task :prepare_all_galleries do
  directory = "./photos/"
  Dir.new(directory).entries.each do |gallery|
    path = directory + gallery
    next if gallery == '.' || gallery == '..' || !File.directory?(path)
    _prepare_photo_gallery path
  end
end

desc 'Start local webserver to test the static site. Opens a new browser window to the home index. Server continues running, so multiple builds can be done and just requires a reload in the browser.'
task :serve do
	require 'shell'
	shell = Shell.new
	shell.pushd '_site'
	shell.system('python -m SimpleHTTPServer 4000 --bind localhost &')
	puts 'Loading site...'
	sleep 3
	shell.system('open http://localhost:4000')
end

desc 'Stop the local webserver.'
task :endserve do
	sh 'killall Python'
end

desc 'Push the git repo to remote and sync the compiled site to S3.'
task :publish do
  sh 'git push origin'
  sh 'aws s3 sync _site/ s3://armcknight.com --exclude .git/ --profile default --acl public-read'
end

desc 'Create a new item for _ideas collection.'
task :idea,[:content] do |t, args|
  time = Time.now
  localized_date = time.strftime('%B %-d, %Y')
  localized_time = time.strftime('%l:%M %p %Z').strip
  utc = time.utc
  utc_date = utc.strftime('%F')
  utc_time = utc.strftime('%T').gsub(':', '-')
  filename = utc_date + '-' + utc_time + '.html'
  ideas_dir = '_ideas'
  if !Dir.exist?(ideas_dir) then
    sh "mkdir #{ideas_dir}"
  end
  path = [ideas_dir, filename].join('/')
  _inject_values_into_template 'idea', path, {
    'idea_content' => args[:content].to_s,
    'created_date' => localized_date,
    'created_time' => localized_time
  }
end

def _inject_values_into_template template_filename, filename, mapping
  File.open("_templates/#{template_filename}", 'r') do |template_file|
    File.open(filename, 'w', File::CREAT) do |file|
      file << mapping.inject(template_file.read) do |filled, (item, value)|
        filled.gsub("$$#{item}$$", value.to_s)
      end
      file.chmod(0644)
    end
  end
end

def _prepare_photo_gallery input_dir
  require 'date'
  require 'open3'

  # get and check some required paths
  raise "Image directory does not exist (#{input_dir})." unless Dir.exist?(input_dir)
  
  # lowercase filename extensions
  Dir.new(input_dir).each do |image|
    # downcase extensions
    if image.include?('.JPG') then
      next if image == '.' || image == '..' || image == '.DS_Store'
      puts "Downcasing extension for #{image}"
      url = "#{input_dir}/#{image}"
      new_url = url.gsub('.JPG', '.jpg')
      File.rename(url, new_url)
    end
  end

  # generate the yaml front matter for the album
  album_id = input_dir.split('/').last
  album_yaml_url = "#{input_dir}/../../_albums/#{album_id}.html"
  if File.exist?(album_yaml_url) then
    album_front_matter = IO.readlines(album_yaml_url)
    album_name = album_front_matter[1].gsub('name: ', '').chomp[1..-2]
    album_description = album_front_matter[2].gsub('description: ', '').chomp[1..-2]
    cover_image_url = album_front_matter[3].gsub('cover_image_url: ', '').chomp
    puts "Album front matter exists...\nread album name: \"#{album_name}\"\ndescription: \"#{album_description}\"\ncover image thumbnail url: \"#{cover_image_url}\""
  else
    puts "Enter album name: "
    album_name = STDIN.gets.chomp
    puts "Enter album description: "
    album_description = STDIN.gets.chomp
    puts "Enter album cover image thumbnail url: "
    cover_image_url = STDIN.gets.chomp
  
    _inject_values_into_template 'album', album_yaml_url, {
      'album_name' => album_name,
      'album_description' => album_description,
      'cover_image_url' => cover_image_url,
    }
  end

  # create the slideshow
  slideshow_dir = "#{input_dir}/slideshow"
  sh "mkdir #{slideshow_dir}" unless Dir.exist?(slideshow_dir)
  slideshow_template = "#{slideshow_dir}/index.html"
  hash = {
    'album_id' => album_id,
    'album_name' => album_name,
    'album_description' => album_description,        
  }
  puts hash
  _inject_values_into_template 'slideshow', slideshow_template, hash

  # move images to img/ subdirectory
  image_subdirectory = "#{input_dir}/img"
  if Dir.exist?(image_subdirectory) then
    puts "img/ subdirectory already exists, working with whatever images are in there."
  else
    sh "mkdir #{image_subdirectory}"
    sh "mv #{input_dir}/*.jpg #{image_subdirectory}"
  end

  # gather information about the images to display and generate thumbnails
  images = Hash.new
  Dir.new(image_subdirectory).each do |image|
    next if image == '.' || image == '..' || image == '.DS_Store' || image.include?('thumbnail')
    url = "#{image_subdirectory}/#{image}"
    
    image_description = `exiftool -p '$description' #{url}`
    if image_description == "" then
      Open3.popen3("open #{url}") do |i,o,e,t|
        puts "Enter description for #{url}:"
        image_description = STDIN.gets.chomp.gsub('"', '\'')
        if image_description != "" then
          sh "exiftool -description=\"#{image_description}\"  -overwrite_original #{url}"
        end
        exit_status = t.value
      end
    end
    
    timestamp = `exiftool -p '$modifydate' #{url}`
    date = DateTime.strptime(timestamp, '%Y:%m:%d %H:%M:%S')
    if images[date] != nil then
      puts "Duplicate images at #{date}: #{images[date]} and #{image}."
      exit 1
    end
  
    # generate the thumbnail image now
    thumbnail_url = url.gsub('.jpg', '-thumbnail.jpg')
    if File.exist?(thumbnail_url) then
      puts "Thumbnail already exists for #{image}: #{thumbnail_url}." 
    else
      sh "magick #{url} -resize x100 #{thumbnail_url}" unless File.exist?(thumbnail_url)
    end
    thumbnail_width = `exiftool -p '$imageWidth' #{thumbnail_url}`.strip
    thumbnail_width = 150 if thumbnail_width.to_i < 150
      
    images[date] = { 
      'description' => image_description.strip, 
      'url' => image, 
      'thumbnail_url' => thumbnail_url.split('/').last,
      'thumbnail_width' => thumbnail_width,
    }
  end

  # generate the yaml front matter for each picture
  image_idx = 0
  images.keys.sort.each do |sortedKey|
    image = images[sortedKey]
    url = image['url']
    image_width = image['image_width']
    image_height = image['image_height']
    image_description = image['description']
    thumbnail_url = image['thumbnail_url']
    thumbnail_width = image['thumbnail_width']

    filename = "#{input_dir}/../../_photos/#{album_id}-#{image_idx < 10 ? '0' + image_idx.to_s : image_idx.to_s}.html"
    _inject_values_into_template 'image', filename, {
      'image_idx' => image_idx,
      'url' => url,
      'image_description' => image_description,
      'thumbnail_url' => thumbnail_url,
      'thumbnail_width' => thumbnail_width,
      'date' => sortedKey.strftime("%B %e, %Y"),
      'album_id' => album_id,
    }
  
    image_idx += 1
  end 
  
  # create gallery index.html
  _inject_values_into_template 'gallery', "#{input_dir}/index.html", {
    'album_name' => album_name,
    'album_description' => album_description,
    'album_id' => album_id,
    'cover_image_thumbnail_url' => cover_image_url.gsub('.jpg', '-thumbnail.jpg'),
    'cover_image_url' => cover_image_url,
  }
end 

def _build
	sh 'sass --update css'
	sh 'jekyll build --incremental'
end