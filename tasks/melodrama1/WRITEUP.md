# Melodrama I: Write-up

При подключении на указанные в задании хост и порт нам предлагают ввести токен, после чего нас встречает консольный интерфейс для какого-то редактора СМИ. Очень похоже, что после ввода токена для нас запускают программу, исходный код которой дан в таске.

В условии нас просят найти «тайные заметки для другого СМИ». И действительно, в коде мы видим загрузку какой-то секретной заметки из файла `flag1.txt`:

```c
    articles[0] = malloc(sizeof(article_t));
    articles[0]->secret = 1;
    fscanf(fp, "%s", articles[0]->note);
    articles[0]->length = strlen(articles[0]->note);
    fclose(fp);
```

Мы можем попробовать прочитать её через меню, но ничего не выйдет:

```
Your token: 05b2f3ee9b318a7468467a407491d6ca
  __  __      _           _                            
 |  \/  |    | |         | |                           
 | \  / | ___| | ___   __| |_ __ __ _ _ __ ___   __ _  
 | |\/| |/ _ \ |/ _ \ / _` | '__/ _` | '_ ` _ \ / _` | 
 | |  | |  __/ | (_) | (_| | | | (_| | | | | | | (_| | 
 |_|  |_|\___|_|\___/ \__,_|_|  \__,_|_| |_| |_|\__,_| 
                                                       
  International Twitter mass media agency since 1987.

Hi, editor!
What do you want to do?

 1 - add an article
 2 - suggest an edit
 3 - apply suggested edit
 4 - send to chief editor
 5 - read an article
 6 - delete an article
 7 - exit
> 5
Enter article ID:
0
You can't see this article.
```

Изучая исходный код программы, находим функцию, отвечающую за удаление статьи:

```c
void delete_article() {
    int id;
    puts("Enter article ID:");
    scanf("%d", &id);

    if (id < 0 || id >= ARTICLES || !articles[id]) {
        puts("No such article.");
        return;
    }

    memset(articles[id], NULL, sizeof(article_t*));
}
```

Что же делает эта функция? В начале она читает ID статьи и проверяет её на существование — эта часть совпадает с другими функциями. За удаление отвечает последняя строчка. Мы можем найти описание функции `memset` в [man](https://linux.die.net/man/3/memset) или в интернете. Эта функция заполняет область памяти нужным значением.

Мы действительно устанавливаем в 0 (`NULL == 0`) область памяти размером с указатель. Но, к сожалению, не в том месте. Когда мы передаем `articles[i]` в качестве адреса, в аргумент попадает не местоположение этой ячейки, а её значение. А поскольку в массиве `articles` хранятся указатели на статьи, первым аргументом функции оказывается область памяти самой заметки.

Что ж, осталось разобраться с теми плачевными последствиями, к которым этот баг приводит. Для этого рассмотрим содержимое самой структуры `articles_t`:

```c
typedef struct {
    int secret;
    int length;
    /* 140 chars limit + zero byte */
    char note[141];
    char signature[64];
} __attribute__((packed)) article_t;
```

В памяти поля структуры распологаются в том же порядке, что и в её описании. Более того, `__attribute__((packed))` гарантирует, что между полями структуры не будет промежутков.

`sizeof(articles_t*)` будет равен либо четырём, либо восьми байтам, в зависимости от того, какая операционная система у нас установлена¹. Первые четыре байта структуры² содержат поле `secret` типа `int`. Какое удачное совпадение: именно это поле нас и интересует — во всех публичных заметках оно равно нулю:

```c
    if (articles[id]->secret) {
        puts("You can't see this article.");
        return;
    }
```

Получаем следующий простой [алгоритм](exploit.py) получения флага:

1. Удаляем заметку 0.
2. Читаем заметку 0.

Флаг: **ugra_nullptr_is_a_zero_ab198875fc77**

<hr/>

¹ На сервере установлена 64-битная операционная система, поэтому указатель имеет размер 8 байт. Таким образом, стирается и поле `length`. Тем не менее, нам кажется, что инконсистетность в этом поле в меньшую сторону не приводит к каким-то ошибкам.

² Стандарт C99 (ISO/IEC 9899:1999) устанавливает, что значение типа `int` может иметь любой размер не менее 16 бит. Однако, можно самостоятельно убедиться, что все современные реализации GCC (в комментарии в первой строке указано, что использовался именно этот компилятор) используют 32 бита для хранения данных этого типа.
