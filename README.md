After roughly 2 days struggle (yes, struggle and scrach head ) , finally Successfully build OpenJDK9 on Mac 10.14(Mojave)   
with Xcode 10.1 using Clang 10.0.  

Also , import the project to Xcode IDE for debug (really cool ).  

**Why build OpenJDK9 on Mac ?**   

OpenJDK8 may need gcc compiler and has version required like 4.9.  
Xcode only have gcc 4.2.1 compatability due to license issue.  
Another concern is I want to browse code in Xcode IDE and debug as well, so use Clang would be a better option as it is 
supported by Apple.

--------------------
**Steps**  

1. Download source code from https://github.com/unofficial-openjdk/openjdk.  
This is essential , at least for me.  I have tried the other github source for openjdk and only succeed with this one.  

   Then checkout the openjdk9 code :  
   git checkout origin/openjdk9  -b jdk9  

2. Dependency setup.  
brew install freetype   #for font image
brew install ccache  #for compiler cache to speed up compilation.  

3. go to the git repo and run configure:  
bash configure --with-target-bits=64 --with-freetype=/usr/local/Cellar/freetype/2.9.1 --enable-ccache --with-jvm-variants=server,client --with-boot-jdk-jvmargs="-Xlint:deprecation -Xlint:unchecked" --disable-zip-debug-info --disable-warnings-as-errors --with-debug-level=slowdebug \
--with-extra-cxxflags='-stdlib=libc++' \
 --with-extra-cflags='-stdlib=libc++ ' \
--with-extra-ldflags='-stdlib=libc++' 2>&1 | tee configure_mac_x64.log

--with-freetype=/usr/local/Cellar/freetype/2.9.1  to add freetype location as the configure script can not find that.  

-disable-warnings-as-errors to disable the feature reporting warnings as error.

--with-extra-cxxflags[cflags,ldflags] : This is to use libc++ which is the replacment of libstdc++ (in GCC) for Clang.
This is **required**, otherwise you may see "<new> file not found" or header not found, etc .


**Before you run "make" , need to change below codes :**
 *  hotspot/src/share/vm/runtime/perfData.cp 
   comment the "delete p;" in the "destroy()" method . Otherwise , you may see the same issue as :  
   https://stackoverflow.com/questions/50678467/building-openjdk-9-on-mac-os  
   https://bugs.openjdk.java.net/browse/JDK-8204325
   
 ```
 void PerfDataManager::destroy() {

  if (_all == NULL)
    // destroy already called, or initialization never happened
    return;

  for (int index = 0; index < _all->length(); index++) {
    PerfData* p = _all->at(index);
    delete p;
  }

  delete(_all);
  delete(_sampled);
  delete(_constants);

  _all = NULL;
  _sampled = NULL;
  _constants = NULL;
}
```

 *  check this https://bugs.openjdk.java.net/browse/JDK-8187787
```
rraghavan Rahul Raghavan added a comment - 2017-11-08 00:59
(#08Nov2017) - Now confirmed following are already in place at http://hg.openjdk.java.net/jdk/hs - 

#1. src/hotspot/share/memory/virtualspace.cpp # l585 
  if (base() != NULL) { 

#2. src/hotspot/share/opto/lcm.cpp # l42 
  if (Universe::narrow_oop_base() != NULL) { // Implies UseCompressedOops. 

#3. src/hotspot/share/opto/loopPredicate.cpp # l915 
      assert(rng->Opcode() == Op_LoadRange || iff->is_RangeCheck() || _igvn.type(rng)->is_int()->_lo != NULL, "must be"); 

So closing 8187787 as not a bug
```
 
 4. Run make  
export LANG=C  
make all LOG=debug  2>&1 | tee make_mac_x64.log  

---------
**Import project to Xcode for debug**  
follow the instruction of https://www.zhihu.com/question/52169710  
1. create project and cleanup the auto-generated code.  
2. Add new files ( the git repository)  
3. Product -> Edit Scheme -> 
  in Build , remove the target as we have already build the code  
  in Run->info tab, fill the executable as "java" where you have compiled: 
  in Run->argument tab, fill a dummay java project path and class to debug
4. setup a breakpoint on main.c under jdk/src/java.base/share/native/launcher.

5. click run to play now.  

**[Reference]**   
 https://medium.com/@neerajx86/setup-openjdk-9-build-environment-on-macos-d49609f773e9  
 https://www.jianshu.com/p/ee7e9176632c  
 https://stackoverflow.com/questions/52425766/stdlibc-headers-not-found-error-on-xcode-10  
 https://segmentfault.com/a/1190000005082098  
 https://www.jianshu.com/p/34c8a8c37169  
 https://www.zhihu.com/question/52169710  
 
 
