#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <float.h>
#include <time.h>

/*
Un drone dans l'espace : son identifiant et sa position (x, y, z).
*/
struct Drone {
    int   id;
    float x;
    float y;
    float z;
};

/*
Retourne la distance au CARRÉ entre deux drones.
Pourquoi au carré ? Parce que sqrt() est lente et inutile
tant qu'on fait juste des comparaisons. On ne l'appellera
qu'une seule fois, tout à la fin, pour afficher le résultat.
*/
float calculer_distance_carre(struct Drone* d1, struct Drone* d2) {
    float dx = d1->x - d2->x;
    float dy = d1->y - d2->y;
    float dz = d1->z - d2->z;
    return (dx * dx) + (dy * dy) + (dz * dz);
}

/*
Fonction de comparaison passée à qsort pour trier les drones par X.
On utilise la soustraction plutôt qu'un simple if/else parce que
les flottants peuvent jouer des tours avec les arrondis , c'est
la façon propre de faire ça en C.
*/
int comparer_drones_x(const void* a, const void* b) {
    struct Drone* droneA = (struct Drone*)a;
    struct Drone* droneB = (struct Drone*)b;
    float diff = droneA->x - droneB->x;
    return (diff > 0.0f) - (diff < 0.0f);
}

int main() {
    int N = 10000;

    /* On ne peut pas chercher une paire si on a moins de 2 drones. */
    if (N < 2) {
        printf("Erreur : l'essaim doit contenir au moins 2 drones.\n");
        return -1;
    }

    /*
    Tout l'essaim dans un seul bloc mémoire contigu.
    Le compilateur de sécurité interdit les crochets [], donc
    on navigue uniquement avec l'arithmétique de pointeurs :
    (essaim + i)->champ à la place de essaim[i].champ
    */
    struct Drone* essaim = (struct Drone*)malloc(N * sizeof(struct Drone));
    if (essaim == NULL) {
        printf("Erreur critique : echec de l'allocation memoire.\n");
        return -1;
    }

    /*
    On remplit l'essaim avec des positions aléatoires.
    L'espace simulé fait 100 x 100 x 50 mètres  
    une échelle réaliste pour un essaim de drones en conditions réelles.
    */
    srand((unsigned int)time(NULL));
    for (int i = 0; i < N; i++) {
        (essaim + i)->id = i;
        (essaim + i)->x = (float)rand() / (float)RAND_MAX * 100.0f;
        (essaim + i)->y = (float)rand() / (float)RAND_MAX * 100.0f;
        (essaim + i)->z = (float)rand() / (float)RAND_MAX *  50.0f;
    }

    /*
    On trie tous les drones par coordonnée X croissante.
    C'est le point de départ de toute l'optimisation :
    sans ce tri, on ne peut pas faire le break plus bas.
    */
    qsort(essaim, N, sizeof(struct Drone), comparer_drones_x);

    float         distance_min_carre = FLT_MAX;
    struct Drone* drone_en_danger_1  = NULL;
    struct Drone* drone_en_danger_2  = NULL;

    /*
    Pour chaque drone i, 
    on le compare avec ses voisins j qui suivent dans le tableau trié. 
    Le trick, c'est le break :
    Puisque le tableau est trié par X, 
    dès que l'écart sur X seul est déjà plus grand que la meilleure distance trouvée jusqu'ici,
    inutile d'aller plus loin, tous les drones suivants seront encore plus éloignés.
    On coupe la branche et on passe à i+1.
    C'est ce break qui transforme un O(n²) catastrophique en quelque chose 
    de beaucoup plus raisonnable en pratique.
    */
    for (int i = 0; i < N - 1; i++) {
        struct Drone* drone_actuel = (essaim + i);

        for (int j = i + 1; j < N; j++) {
            struct Drone* drone_suivant = (essaim + j);

            /* si l'écart en X suffit à dépasser le min, on arrête. */
            float diff_x = drone_suivant->x - drone_actuel->x;
            if ((diff_x * diff_x) >= distance_min_carre) {
                break;
            }

            /* Sinon on calcule vraiment la distance 3D et on met à jour si besoin. */
            float dist_carre = calculer_distance_carre(drone_actuel, drone_suivant);
            if (dist_carre < distance_min_carre) {
                distance_min_carre = dist_carre;
                drone_en_danger_1  = drone_actuel;
                drone_en_danger_2  = drone_suivant;
            }
        }
    }

    /* Affichage du résultat ou message d'erreur si quelque chose s'est mal passé. */
    if (drone_en_danger_1 != NULL && drone_en_danger_2 != NULL) {
        printf("ALERTE COLLISION DETECTEE\n");
        printf("Drone #%d : (%.2f, %.2f, %.2f)\n",
               drone_en_danger_1->id,
               drone_en_danger_1->x,
               drone_en_danger_1->y,
               drone_en_danger_1->z);
        printf("Drone #%d : (%.2f, %.2f, %.2f)\n",
               drone_en_danger_2->id,
               drone_en_danger_2->x,
               drone_en_danger_2->y,
               drone_en_danger_2->z);
        printf("Distance de separation : %.4f metres\n", sqrtf(distance_min_carre));
    } else {
        printf("Aucune paire detectee — essaim vide ?\n");
    }

    free(essaim);
    essaim = NULL; /* on ne laisse pas traîner un pointeur mort */

    return 0;
}
