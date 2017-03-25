require "pathname"

root_dir = Pathname.new(__FILE__).dirname
required_tools = %w{
  libncurses5-dev
  gcc-arm-linux-gnueabihf
}
tool_chain = "arm-linux-gnueabihf"

desc "install required tools"
task :install_tools do
  required_tools.each do |r|
    sh "dpkg -s #{ r }" do |ok, status|
      if !ok
        sh "apt-get install #{ r }"
      end
    end
  end
end

desc "update kernel source"
task :update do
  sh "git submodule update"
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

desc "build kernel"
task :build_kernel do
  Dir.chdir("linux") do
    sh "make -j 4 -k ARCH=arm CROSS_COMPILE=#{ tool_chain }- menuconfig"
    sh "make -j 4 -k ARCH=arm CROSS_COMPILE=#{ tool_chain }- bzImage"
    sh "cp arch/arm/boot/zImage #{ root_dir }/kernel-qemu-`make kernelversion`"
  end
end

desc "clean"
task :clean do
  Dir.chdir("linux") do
    sh "make clean"
    sh "git reset --hard HEAD^"
  end
  sh "rm -f linux/.config patch.rej"
end

desc "run the build"
task :build => [ :install_tools, :update, :patch, :make_versatile_defconfig, :config, :build_kernel ]
