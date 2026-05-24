#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/inotify.h>
#include <limits.h>

#define BUF_LEN (1024 * (sizeof(struct inotify_event) + NAME_MAX + 1))

typedef struct {
    uint32_t cookie;
    char nome[NAME_MAX];
} spostamento_t;

spostamento_t ultimo_spostamento = {0};

/* Stampa il tipo di evento */
void stampa_maschera(uint32_t mask) {

    if (mask & IN_CREATE)
        printf("CREAZIONE ");

    if (mask & IN_DELETE)
        printf("ELIMINAZIONE ");

    if (mask & IN_MOVED_FROM)
        printf("SPOSTATO_DA ");

    if (mask & IN_MOVED_TO)
        printf("SPOSTATO_A ");

    if (mask & IN_ISDIR)
        printf("DIRECTORY ");
}

int main(int argc, char *argv[]) {

    /* Controllo argomenti */
    if (argc < 2) {

        printf("uso: %s directory\n", argv[0]);

        return 1;
    }

    /* Creazione istanza inotify */
    int fd = inotify_init();

    if (fd < 0) {

        perror("inotify_init");

        return 1;
    }

    /* Aggiunta watch sulla directory */
    int wd = inotify_add_watch(
        fd,
        argv[1],

        IN_CREATE |
        IN_DELETE |
        IN_MOVED_FROM |
        IN_MOVED_TO
    );

    if (wd < 0) {

        perror("inotify_add_watch");

        return 1;
    }

    printf("Monitoraggio directory: %s\n", argv[1]);

    char buffer[BUF_LEN];

    /* Loop infinito */
    while (1) {

        int lunghezza = read(fd, buffer, BUF_LEN);

        if (lunghezza <= 0)
            continue;

        for (char *ptr = buffer;
             ptr < buffer + lunghezza; ) {

            struct inotify_event *evento =
                (struct inotify_event *) ptr;

            stampa_maschera(evento->mask);

            if (evento->len > 0)
                printf("nome=%s ", evento->name);

            /* Prima parte rename/move */
            if (evento->mask & IN_MOVED_FROM) {

                ultimo_spostamento.cookie =
                    evento->cookie;

                strcpy(
                    ultimo_spostamento.nome,
                    evento->name
                );
            }

            /* Seconda parte rename/move */
            if (evento->mask & IN_MOVED_TO) {

                if (evento->cookie ==
                    ultimo_spostamento.cookie) {

                    printf(
                        "RINOMINA %s -> %s",
                        ultimo_spostamento.nome,
                        evento->name
                    );
                }
            }

            printf("\n");

            ptr += sizeof(struct inotify_event)
                   + evento->len;
        }
    }

    close(fd);

    return 0;
}
