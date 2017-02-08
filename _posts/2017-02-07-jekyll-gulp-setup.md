---
  title: Настройка Gulp для работы с Jekyll
  date: 2017-02-07 14:04+0300
  description: Разбираем основные моменты в настройке gulp при работе с Jekyll. Работаем с модулем Child Process
  category: [jekyll, youtube, разработка]
  poster: /img/posters/poster17.jpg
---
В дополнении к видеоролику о настройке Gulp хотел бы еще опубликовать статью, по мотивам которой и снимал скринкаст. Своего рода - текстовый вариант записи. Также приведу все примеры кода для удобства использования.

### Gulp, зачем?

Я надеюсь из видео вполне ясна была моя идея. **Gulp** - это удобно, современно, качественно и быстро. Почти всё, что может пригодится в небольшом проекте на самом деле умеет делать и сам Jekyll без посторонней помощи. Поэтому эта задачка просто на будущее. С этим сборщиком работают сейчас очень многие, было бы неправильно обойти его стороной, тем более что с ним все таки будет удобнее верстать) 

### Модуль child_process

Здесь я опущу вопросы установки и первого запуска gulp. Подобной информации везде предостаточно, в том числе в официальной документации. Сразу хотелось бы перейти к работе с модулем **child_process**.

Итак, как известно, к помощи этого модуля необходимо обратиться тогда, когда нам необходима работа в командной строке посредством Gulp'a. Запустить исполняемый файл, поработать с файловой системой через консоль, ну или касательно нашего случая - выполнить команду `jekyll build` для сборки сайта.

В документации содержится подробная информация обо всех методах класса Child Process, нам наиболее подходят два метода - это [child_process.spawn][spawn] и [child_process.exec][exec]. 

Получилось выполнить задачу одинаково в обоих вариантах. Необходимо не забывать возвращать колбэк в задачу галпу, потому что в этой задаче нам критичен момент завершения дочернего процесса. В случае метода `spawn` колбэк вызываем после того как повесим обработчик события `close` на поток, а в случае с методом `exec` глобальный колбэк размещаем в теле другого колбэка, которые вызывается также после завершения дочернего процесса, но тут уже обработчик события нам не нужен. Приведу далее код, который получился в видео:

{% highlight javascript %}

    const gulp = require('gulp');
    const spawn = require('child_process').spawn;
    const exec = require('child_process').exec;

     gulp.task('jekyll:build', function (done) {
         return spawn('jekyll', ['build'], {
             shell: true,
             stdio: 'inherit'
         }).on('close', done);
     });
    
    gulp.task('jekyll:build', function (done) {
        exec('jekyll build', function (error, stdout, stderr) {
            if (error) {
                console.log(`exec error ${error}`);
                return;
            }
            console.log(`exec stdout ${stdout}`);
            console.log(`exec stderr ${stderr}`);
            done();
        });
    });
{% endhighlight %}

В интернете вы можете встретить обсуждения на тему того, что используя метод `exec` можно переполнить буфер для `stdout` и поток аварийно завершится и поэтому лучше использовать метод `spawn`. По умолчанию свойство `MaxBuffer = 200Kb`, при желании, конечно его даже можно изменить в опциях при запуске дочернего процесса. 

Однако, сделаю и другие замечания. Насколько я правильно понял [документацию][maxBuffer], свойство `MaxBuffer` применимо для любого потока `child.stdout` или `child.stderr`. А эти потоки в свою очередь доступны у любого экземпляра класса Child Process, поэтому если вы и переполните все же буфер какого либо из этих потоков, то дочерний процесс должен завершиться в обоих случаях независимо exec или spawn метод вы применяли. А переполнить поток нашей конкретной задачей просто нереально, на видео можно увидеть момент, что длина потока stdout от команды jekyll build порядка 334b. 

Поэтому думаю оба метода применимы на равных условиях, ну или же какой заработает на windows))

### Browser-sync

Популярный удобный модуль. Помимо того, что с помощью него мы автоматически будем обновлять браузер при изменении исходников, также этот модуль позволяет наблюдать за изменениями с других устройств и полностью синхронизирует перемещения и действия на странице сайта. Пример - пишем сайт и смотрим сразу как он отображается на лэптоп, десктоп и смартфоне. 

{% highlight javascript %}

    const bs = require('browser-sync').create();
    
    gulp.task('browser-sync', ['jekyll:build'], function () {
        bs.init({
            server: {
                baseDir: "_site"
            }
        });
    });

{% endhighlight %}

### gulp serve

Такой командой, аналогичной `jekyll serve`, мы назвали итоговую задачу, которая объединила в себе все предыдущие.
Предварительно создав еще парочку простых задач для слежения за файлами и пересборки проекта. 

{% highlight javascript %}

    gulp.task('jekyll:rebuild', ['jekyll:build'], function () {
        bs.reload();
    });

    gulp.task('watch', function () {
        gulp.watch('*.html', ['jekyll:rebuild']);
    });

    gulp.task('serve', ['browser-sync', 'watch']);
    
{% endhighlight %}

Подобный пример имеет достаточно простую логику, и он был бы проще если бы не пришлось разбираться в child_process)) Этот код послужит отправной точкой для дальнейшей работы с gulp в нашем проекте. 

### Итоговый gulpfile.js

<script src="https://gist.github.com/kiviok/a5dc711f5ae674e3fe874a4bf6f288e9.js"></script>

<iframe width="560" height="315" src="https://www.youtube.com/embed/O5YKXvjDEkI?list=PLiQQbsp51_qRqVEsKWXnG2WjhimnKcgOU" frameborder="0" allowfullscreen></iframe>

[spawn]: https://nodejs.org/dist/latest-v7.x/docs/api/child_process.html#child_process_child_process_spawn_command_args_options
[exec]: https://nodejs.org/dist/latest-v7.x/docs/api/child_process.html#child_process_child_process_exec_command_options_callback
[maxBuffer]: https://nodejs.org/dist/latest-v7.x/docs/api/child_process.html#child_process_maxbuffer_and_unicode
