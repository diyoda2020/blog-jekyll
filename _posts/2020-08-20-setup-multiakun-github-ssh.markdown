---
layout: post
title:  "Setup Multi Akun Github SSH"
date:   2020-08-19 11:00:04 +0700
categories: Github
---
Biasanya kita akan mengmankan akun github kita dengan menggunkan SSH. Hal ini tidak jadi masalah apabila kita hanya mempunyai 1 (satu) akun saja. Namun apabila kita mempunyai banyak akun, karena tiap github harus memiliki key unik tersendiri. Apabila key tersebut sdh dipakai kaun lain sehingga tidak unik lagi, maka biasanya akan muncul error :  

{% highlight ruby %}
Error: Permission to user/repo denied to user/other-repo
{% endhighlight %}

terus bagaimana caranya agar tiap-tiap akun mempunyai key unik tersendiri ?,di sini saya memberikan tutorial membuat key di sistem operasi Ubuntu 18.04/ Linux untuk 2 (dua) akun.
pertama - tama kita buat key dengan sintaks sebagi berikut :

{% highlight ruby %}
$> ssh-keygen -t rsa -f ~/.ssh/akun1deploy_rsa -C "akun1 deploy"

// setelah proses selesai, jalankan :

$> ssh-keygen -t rsa -f ~/.ssh/akun2deploy_rsa -C "akun2 deploy"
{% endhighlight %}

maka kita akan mempunyai 2 (dua) pasang key baru, yaitu :

{% highlight ruby %}

-rw-------  1 rifan75 rifan75 1679 Agu 20 09:52 akun1deploy_rsa
-rw-r--r--  1 rifan75 rifan75  399 Agu 20 09:52 akun1deploy_rsa.pub
-rw-------  1 rifan75 rifan75 1679 Agu 20 09:52 akun2deploy_rsa
-rw-r--r--  1 rifan75 rifan75  396 Agu 20 09:52 akun2deploy_rsa.pub

{% endhighlight %}

Lalu kita buka akun1 github, dan masuk ke bagian setting, dengan cara mengklik avatar maka akan muncul beberapa menu, yang salah satunya adalah setting, setelah itu kita masuk ke menu "SSH and GPG Keys". kita add key dengan mengcopy isi file  akun1deploy_rsa.pub untuk akun 1. Lalu buka lagi untuk akun 2 dan lakukan juga sama seperti keterangan di atas.

Untuk menggunakan key tersebut, kita buat dahulu file config di folder .ssh dengan isi sebagai berikut :

{% highlight ruby %}

$> cd ~/.ssh
$ .ssh> touch config
$ .ssh> gedit config

// lalu isikan file tersebut dengan :

Host github-akun1
HostName github.com
User git
IdentityFile ~/.ssh/akun1deploy_rsa

Host github-akun2
HostName github.com
User git
IdentityFile ~/.ssh/akun2deploy_rsa

{% endhighlight %}

Setelah itu tiap kita buat repository baru, kita bisa menggunakan key tersebut untuk push repo kita ke akun github, dengan cara :

{% highlight ruby %}

$> git remote set-url origin github-akun1:username/repository.git

// username diganti dengan username akun anda, begitu juga dengan repository.

{% endhighlight %}

begitu juga untuk akun 2, sama perlakuannya.

Demikian tutorial dari saya, semoga bermanfaat.