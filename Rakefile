require "pathname"
require "uri"
require "ruby_expect"
require "logger"
require "shellwords"

root_dir = Pathname.new(__FILE__).dirname
required_tools = %w{
  bc
  exuberant-ctags
  gcc
  gcc-arm-linux-gnueabihf
  libncurses5-dev
  libssl-dev
  make
}
@default_commit = "rpi-4.4.y"
@commit = ENV["RPI_KERNEL_COMMIT"] ? ENV["RPI_KERNEL_COMMIT"] : @default_commit
tool_chain = "arm-linux-gnueabihf"
n_processor = `getconf _NPROCESSORS_ONLN`.chomp
raspbian_image_url = URI("https://downloads.raspberrypi.org/raspbian/images/raspbian-2017-03-03/2017-03-02-raspbian-jessie.zip")
raspbian_image_name = raspbian_image_url.path.split("/").last.split(".").first # "2017-03-02-raspbian-jessie"

desc "install required tools"
task :install_tools do
  required_tools.each do |r|
    tools = []
    sh "dpkg -s #{ r } >/dev/null" do |ok, status|
      if !ok
        tools << r
      end
    end
    sh "sudo apt-get install #{ tools.join(' ') }" if tools.length > 0
  end
end

desc "update kernel source"
task :update do
  sh "git submodule update"
end

desc "checkout a commit"
task :checkout do
  Dir.chdir("linux") do
    sh "git checkout #{ @commit }"
  end
end

desc "apply patches"
task :patch do
  patches = []
  Dir.glob("configs/*.patch") do |c|
    patches << Pathname.new(c)
  end
  if patches.length > 0
    Dir.chdir "linux" do
      sh "cat #{ patches.join(" ") } | patch --reject-file=#{ root_dir + 'patch.rej' } -p1"
    end
  end
end

desc "make versatile_defconfig"
task :make_versatile_defconfig do
  Dir.chdir "linux" do
    sh "make ARCH=arm versatile_defconfig"
  end
end

desc "create .config"
task :config do
  Dir.glob("configs/*.config") do |c|
    sh "cat #{ c } >> linux/.config"
  end
end

desc "compile kernel"
task :compile_kernel do
  Dir.chdir("linux") do
    sh "make -j #{ n_processor } -k ARCH=arm CROSS_COMPILE=#{ tool_chain }- menuconfig"
    sh "make -j #{ n_processor } -k ARCH=arm CROSS_COMPILE=#{ tool_chain }- bzImage"
    sh "cp arch/arm/boot/zImage #{ root_dir }/kernel-qemu-`make kernelversion`-#{ @commit }"
  end
end

desc "fetch raspian image"
task :fetch_raspian_image do
  zip = raspbian_image_name + ".zip"
  if !File.exists?(zip)
    sh "wget -O images/#{ zip } #{ raspbian_image_url }"
  end
end

desc "extract the image"
task :extract_image do
  image = raspbian_image_name + ".img"
  if !File.exists?("images/#{ image }")
    sh "unzip images/#{ raspbian_image_name }.zip -d images"
  end
end

def kernel_name
  Dir.chdir("linux") do
    "kernel-qemu-" + `make kernelversion`.chomp + "-" + @commit
  end
end

desc "modify the image"
task :modify_image do
  offset = 0
  Dir.chdir("images") do
    offset = `fdisk -l #{ raspbian_image_name }.img | grep #{ raspbian_image_name }.img2`.chomp.split(/\s+/)[1]
  end
  begin
    sh "sudo mount -o offset=#{ offset.to_i * 512 } images/#{ raspbian_image_name }.img /mnt"
    sh "sudo cp /dev/null /mnt/etc/ld.so.preload"
    sh "sudo cp files/90-qemu.rules /mnt/etc/udev/rules.d"
    sh "sudo sed -i'' -e 's|^/dev/mmcblk0|#/dev/mmcblk0|' /mnt/etc/fstab"
    sh "sudo patch -d /mnt -b -p1 < files/raspi-config.patch"
  ensure
    sh "sudo umount /mnt"
  end
end

desc "expand root file system"
task :expand_root do
  sh "qemu-img resize images/#{ raspbian_image_name }.img +4G"

  qemu_flags = "-cpu arm1176 -m 256 -M versatilepb -no-reboot -serial stdio" +
    " -display none" +
    " -append 'root=/dev/sda2 panic=1 rootfstype=ext4 rw console=ttyAMA0 console=ttyS0'"
  command = "qemu-system-arm #{ qemu_flags } -kernel #{ root_dir }/#{ kernel_name } -hda images/#{ raspbian_image_name }.img"
  puts command

  exp = RubyExpect::Expect.spawn(command)
  logger = Logger.new(STDOUT)
  logger.datetime_format = ""
  exp.logger = logger
  exp.procedure do
    each do
      expect(/^raspberrypi login:/) do
        send "pi"
      end

      expect(/^Password:/) do
        send "raspberry"
      end
      expect(/pi@raspberrypi/) do
        send "sudo su"
      end
      prompt = /root@raspberrypi/

      commands = [
        "sudo su",
        "cat /etc/fstab",
        "mount",
        "mount /dev/sda1 /boot",
        "cat /etc/udev/rules.d/90-qemu.rules",
        "update-rc.d ssh enable",
        "unlink /dev/root;ln -s mmcblk0p2 /dev/root",
        "cd /",
      ]
      commands.each do |c|
        expect prompt do
          send c
        end
      end

      expect prompt do
        send "raspi-config"
      end

    end

    # TODO handle Errno::EIO
    exp.interact
  end
end

desc "clean"
task :clean do
  Dir.chdir("linux") do
    sh "make clean"
    sh "rm -f #{ root_dir }/kernel-qemu-`make kernelversion`"
  end
  sh "rm -f images/#{ raspbian_image_name }.img"
  sh "rm -f linux/.config patch.rej"
end

desc "run the build"
task :build => [ :install_tools, :build_kernel, :build_image ]

desc "build kernel"
task :build_kernel => [ :checkout, :patch, :make_versatile_defconfig, :config, :compile_kernel ]

desc "build image"
task :build_image => [ :extract_image, :modify_image, :expand_root ]
