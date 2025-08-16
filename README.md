#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <time.h>
#include <signal.h>

#define PORT 8080
#define BUFFER_SIZE 4096
#define BACKLOG 10

pthread_mutex_t lock;
int server_fd;

void handle_sigint(int sig) {
    printf("\n[!] Shutting down server...\n");
    close(server_fd);
    pthread_mutex_destroy(&lock);
    exit(0);
}

typedef struct {
    int socket;
    struct sockaddr_in address;
} client_info;

void log_request(const char *client_ip, const char *path) {
    pthread_mutex_lock(&lock);
    FILE *log = fopen("log.txt", "a");
    if (log) {
        time_t now = time(NULL);
        fprintf(log, "[%s] %s requested %s\n", ctime(&now), client_ip, path);
        fclose(log);
    }
    pthread_mutex_unlock(&lock);
}

const char *get_content_type(const char *path) {
    if (strstr(path, ".css")) return "text/css";
    return "text/html";
}

const char *get_response_body(const char *path) {
    if (strcmp(path, "/") == 0 || strcmp(path, "/index") == 0) {
        return
            "<!DOCTYPE html>"
            "<html><head><title>Home</title><style>"
            "body { font-family:sans-serif; background:#f4f4f4; text-align:center; padding:50px; }"
            "a { text-decoration:none; color:#333; font-size:18px; }"
            "</style></head><body>"
            "<h1>Welcome to My C Web Server</h1>"
            "<p>This is the <b>home page</b> served dynamically.</p>"
            "<p><a href='/about'>About</a></p>"
            "</body></html>";
    } else if (strcmp(path, "/about") == 0) {
        return
            "<!DOCTYPE html>"
            "<html><head><title>About</title><style>"
            "body { font-family:sans-serif; background:#fff; text-align:center; padding:50px; }"
            "a { text-decoration:none; color:#0077cc; }"
            "</style></head><body>"
            "<h1>About This Project</h1>"
            "<p>This web server was created in C for an OS Lab project.</p>"
            "<p><a href='/'>Back to Home</a></p>"
            "</body></html>";
    } else {
        return
            "<!DOCTYPE html>"
            "<html><head><title>404 Not Found</title><style>"
            "body { font-family:sans-serif; background:#fee; text-align:center; padding:50px; }"
            "</style></head><body>"
            "<h1>404 - Page Not Found</h1>"
            "<p>The requested page does not exist.</p>"
            "<p><a href='/'>Go Home</a></p>"
            "</body></html>";
    }
}

int is_valid_path(const char *path) {
    return strcmp(path, "/") == 0 || strcmp(path, "/index") == 0 ||
           strcmp(path, "/about") == 0;
}

void *handle_client(void *arg) {
    client_info *info = (client_info *)arg;
    int client_socket = info->socket;
    struct sockaddr_in addr = info->address;
    free(info);

    char client_ip[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &addr.sin_addr, client_ip, INET_ADDRSTRLEN);

    char buffer[BUFFER_SIZE] = {0};
    ssize_t bytes_read = read(client_socket, buffer, BUFFER_SIZE - 1);
    if (bytes_read <= 0) {
        perror("[-] Read failed or client disconnected");
        close(client_socket);
        return NULL;
    }

    char method[8], path[1024];
    sscanf(buffer, "%s %s", method, path);

    if (strcmp(method, "GET") != 0) {
        const char *error = "HTTP/1.1 405 Method Not Allowed\r\n\r\n";
        write(client_socket, error, strlen(error));
        close(client_socket);
        return NULL;
    }

    log_request(client_ip, path);

    const char *body = get_response_body(path);
    const char *content_type = get_content_type(path);
    int status_code = is_valid_path(path) ? 200 : 404;

    char header[512];
    snprintf(header, sizeof(header),
             "HTTP/1.1 %d %s\r\n"
             "Content-Type: %s\r\n"
             "Content-Length: %zu\r\n"
             "Connection: close\r\n"
             "Server: EmbeddedCServer/1.0\r\n\r\n",
             status_code, status_code == 200 ? "OK" : "Not Found",
             content_type, strlen(body));

    write(client_socket, header, strlen(header));
    write(client_socket, body, strlen(body));

    printf("[✓] %s requested %s (%d)\n", client_ip, path, status_code);
    close(client_socket);
    return NULL;
}

int main() {
    int client_fd;
    struct sockaddr_in address;
    socklen_t addrlen = sizeof(address);

    signal(SIGINT, handle_sigint);
    pthread_mutex_init(&lock, NULL);

    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd == -1) {
        perror("[-] Socket creation failed");
        exit(EXIT_FAILURE);
    }

    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&address, sizeof(address)) < 0) {
        perror("[-] Bind failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    if (listen(server_fd, BACKLOG) < 0) {
        perror("[-] Listen failed");
        close(server_fd);
        exit(EXIT_FAILURE);
    }

    printf("[+] Server is running on port %d\n", PORT);

    while (1) {
        client_fd = accept(server_fd, (struct sockaddr *)&address, &addrlen);
        if (client_fd < 0) {
            perror("[-] Accept failed");
            continue;
        }

        client_info *info = malloc(sizeof(client_info));
        info->socket = client_fd;
        info->address = address;

        pthread_t thread_id;
        if (pthread_create(&thread_id, NULL, handle_client, info) != 0) {
            perror("[-] Thread creation failed");
            close(client_fd);
            free(info);
            continue;
        }

        pthread_detach(thread_id);
    }

    pthread_mutex_destroy(&lock);
    return 0;
}

