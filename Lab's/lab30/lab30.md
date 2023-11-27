Что такое сокет?
Сокет - специальное псевдоустройство подсистемы STREAMS, используемое для связи между процессами на одной машине или на разных.
  Являются улучшенной версией труб (pipe), которые также позволяют передавать данные между процессами, но обладают рядом недостатков:
    - позволяют передавать данные лишь локально и между двумя процессами (тогда как сокеты позволяют общаться множеству процессов с одним, хранящим серверный сокет (listen(2)))
    - Однонаправленные, тогда как сокеты двунаправленные


Вахалия - 17.10

ПРИ КОМПИЛЯЦИИ НАДО ЗАГРУЖАТЬ ДОПОЛНИТЕЛЬНУЮ БИБЛИОТЕКУ socket:
  gcc -Wall -lsocket file.c

socket(2) - создаёт точку связи и возвращает его дескриптор
  int socket(int domain, int type, int protocol)
  domain:
    AF_UNIX / AF_LOCAL - Unix domain socket
      Unix Domain Socket позволяет связываться процессам через сокет в рамках одной системы. 
      Права доступа к сокетам будут определяться простыми правами доступа файлов
    AF_INET - IPv4
    AF_INET6 - IPv6
  type:
    SOCK_STREAM - двунаправленный сокет для передачи потока байтов
    SOCK_DGRAM - сокеты для передачи пакетов (датаграмм)
  Подробнее о доменах, типах и протоколах смотри man -s 2 socket

bind(2) - привязывает имя сокета с его дескриптором
  int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
  struct sockaddr {
      sa_family_t sa_family; // domain
      char        sa_data[14]; // socket name
  }; - общая версия структуры
  struct sockaddr_un {
      sa_family_t sun_family;               // AF_UNIX
      char        sun_path[UNIX_PATH_MAX];  // pathname
  }; - версия для работы с доменом AF_UNIX

listen(2)
  int listen(int sockfd, int backlog);
  Помечает сокет по дескриптору sockfd как пассивный, то есть сокет, который будет принимать входящие подключения через accept(2)
    Иначе говоря, регистрирует сокет как серверный
  backlog показывает максимальную длину очереди ожидаемых подключений от клиентов
  Перед использованием listen должны быть вызваны socket(2) для создания сокета и bind(2) для возможности обращаться к сокету извне

accept(2)
  int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
  Принимает первый в очереди запрос на соединение с сокетом sockfd и возвращает дескриптор на новый сокет, который не будет в состоянии прослушивания и будет связан с обрабатываемым клиентом
  Если передать указатели вторым и третьим аргументами, они будут заполнены данными созданного сокета-клиента
  Если очередь запросов пуста и сокет не отмечен как неблокирующий, процесс будет заблокирован, пока не появится запрос к сокету. Если сокет неблокирующий, то accept завершится с ошибкой EAGAIN или EWOULDBLOCK
  Вместо ожидания в блоке можно использовать select(2) или poll(2), которые сообщат о появлении запроса в очереди

connect(2)
  int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
  Соединяет дескриптор сокета sockfd с адресом, определённым в структуре addr

send(2)
  *Тут не используется, но имеет смысл изучить*

recv(2)
  *Тут не используется, но имеет смысл изучить*

read/write и другие базовые IO операции для сокетов работают через драйвера

как работает atexit()? В частности, как он работает, если в конце функции у нас return, а не exit?
*/


# SERVER.C

    #include <unistd.h>
    #include <stdio.h>
    #include <sys/un.h>
    #include <ctype.h>
    #include <stdlib.h>
    #include <sys/socket.h>
    #include <string.h>
    #include <signal.h>
    #include <locale.h>
    char *path = "./socket";

    void sig_handler(int signo)
    {
        unlink(path);
        exit(0);
    }
    void unlinker(){
        unlink(path);
    }
    int main(int argc, char *argv[]) {
    setlocale(LC_ALL, getenv("LANG"));
        signal(SIGINT, sig_handler);
        struct sockaddr_un address;
        char buffer[100];
        int descriptor, client;
        ssize_t reader;
        if ((descriptor = socket(AF_UNIX, SOCK_STREAM, 0)) == -1) {
            perror("error with socket");
            return -1;
        }
        memset(&address, 0, sizeof(address));
        address.sun_family = AF_UNIX;
        strncpy(address.sun_path, path, sizeof(address.sun_path)-1);
        if (bind(descriptor, (struct sockaddr*)&address, sizeof(address)) == -1) {
            perror("error while binding");
            return -1;
        }
        if (atexit(unlinker)!=0){
            perror("error while atexit");
            return -1;
        }
        if (listen(descriptor, 5) == -1) {
            perror("error while listening");
            return -1;
        }
        if ((client = accept(descriptor, NULL, NULL)) == -1) {
            perror("error while accepting");
            close(descriptor);
            return -1;
        }
        unlink(path);
        while ((reader=read(client,buffer,sizeof(buffer))) > 0){
            for(int i = 0; i < reader; i++){
                putchar(toupper(buffer[i]));
            }
        }
        if (reader == -1) {
            perror("error while reading");
            return -1;
        }
        else if (reader == 0){
            close(client);
            close(descriptor);
            return 0;
        }
    }

# CLIENT.C

    #include <sys/un.h>
    #include <sys/socket.h>
    #include <stdlib.h>
    #include <stdio.h>
    #include <string.h>
    #include <unistd.h>

    char *path = "./socket";

    int main(int argc, char *argv[]) {
        struct sockaddr_un address;
        char buffer[100];
        int descriptor;
        ssize_t reader;
        if ( (descriptor = socket(AF_UNIX, SOCK_STREAM, 0)) == -1) {
            perror("error in socket");
            return -1;
        }
        memset(&address, 0, sizeof(address));
        address.sun_family = AF_UNIX;
        strncpy(address.sun_path, path, sizeof(address.sun_path)-1);
        if (connect(descriptor, (struct sockaddr*)&address, sizeof(address)) == -1) {
            perror("error while connecting");
            return -1;
        }
        memset(buffer, 0, sizeof(buffer));
        while( (reader=read(STDIN_FILENO, buffer, sizeof(buffer))) > 0){
            if (write(descriptor, buffer, reader) != reader && reader == -1) {
                perror("error while writing");
                return -1;
            }
        }
        close(descriptor);
        return 0;
    }


