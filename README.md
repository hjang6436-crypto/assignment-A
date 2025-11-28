#define _CRT_SECURE_NO_WARNINGS
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <math.h> // for radix sort

// --- 1. 정의 및 전역 변수 설정 ---
#define MAX_RECORDS 200000 // 최대 레코드 수, 파일 크기에 따라 조정 가능
#define MAX_NAME_LEN 50
#define MAX_LINE_LEN 200
#define REPETITIONS 1
#define FILENAME "students.csv"

// 측정 변수: 전역 변수로 관리하여 정렬 함수 내에서 접근
long long comparisons = 0; // 비교 횟수
long long moves = 0;       // 데이터 이동/교환 횟수 (메모리 사용량 간접 측정)

typedef struct {
    int id;
    char name[MAX_NAME_LEN];
    char gender;
    int korean;
    int english;
    int math;
    int total; // 합산 점수
} Student;

// --- 2. 데이터 로드 및 초기화 함수 ---

// 사용자 제공 함수 수정: total 필드 계산 추가
Student* load_students(const char* filename, int* out_count) {
    FILE* fp = fopen(filename, "r");
    if (!fp) {
        perror("Failed to open file");
        return NULL;
    }

    char line[MAX_LINE_LEN];
    int capacity = 10000; // 초기 용량 증가
    int count = 0;
    Student* arr = (Student*)malloc(sizeof(Student) * capacity);

    if (!arr) {
        perror("Memory allocation failed");
        fclose(fp);
        return NULL;
    }

    // 첫 줄 헤더 스킵
    if (fgets(line, sizeof(line), fp) == NULL) {
        fclose(fp);
        free(arr);
        *out_count = 0;
        return NULL;
    }

    while (fgets(line, sizeof(line), fp)) {
        if (count >= capacity) {
            capacity *= 2;
            Student* temp = (Student*)realloc(arr, sizeof(Student) * capacity);
            if (!temp) {
                perror("Reallocation failed");
                // 메모리 재할당 실패 시, 현재까지 읽은 데이터를 반환 (메모리 누수 방지)
                fprintf(stderr, "Warning: Reallocation failed, returning partial data.\n");
                fclose(fp);
                *out_count = count;
                return arr;
            }
            arr = temp;
        }

        Student s;
        char temp_line[MAX_LINE_LEN];
        strncpy(temp_line, line, MAX_LINE_LEN - 1);
        temp_line[MAX_LINE_LEN - 1] = '\0';

        char* token = strtok(temp_line, ",");
        if (!token) break;
        s.id = atoi(token);

        token = strtok(NULL, ",");
        if (!token) break;
        strncpy(s.name, token, MAX_NAME_LEN - 1);
        s.name[MAX_NAME_LEN - 1] = '\0';

        token = strtok(NULL, ",");
        if (!token) break;
        s.gender = token[0];

        token = strtok(NULL, ",");
        if (!token) break;
        s.korean = atoi(token);

        token = strtok(NULL, ",");
        if (!token) break;
        s.english = atoi(token);

        token = strtok(NULL, ",");
        if (!token) break;
        // 마지막 토큰은 개행 문자를 포함할 수 있으므로 제거
        s.math = atoi(token);

        // total 필드 계산 및 저장
        s.total = s.korean + s.english + s.math;

        arr[count++] = s;
    }

    fclose(fp);

    // 파일 읽기 완료 → 사용한 만큼만 메모리 딱 맞게 조정
    Student* tight = (Student*)realloc(arr, sizeof(Student) * count);
    if (tight) {
        arr = tight;
    }
    else {
        fprintf(stderr, "경고: 원래 메모리를 사용하여 긴밀한 재할당에 실패했습니다.\n");
    }

    *out_count = count;
    return arr;
}

// --- 3. 비교 함수 (Comparators) ---

// 3.1. ID 기준 오름차순/내림차순
int compare_id_asc(const Student* a, const Student* b) {
    comparisons++;
    return a->id - b->id;
}
int compare_id_desc(const Student* a, const Student* b) {
    comparisons++;
    return b->id - a->id;
}

// 3.2. NAME 기준 오름차순/내림차순
int compare_name_asc(const Student* a, const Student* b) {
    comparisons++;
    return strcmp(a->name, b->name);
}
int compare_name_desc(const Student* a, const Student* b) {
    comparisons++;
    return strcmp(b->name, a->name);
}

// 3.3. GENDER 기준 오름차순/내림차순 (Stable 정렬에만 사용)
// GENDER 오름차순: F < M (0 < 1)
int compare_gender_asc(const Student* a, const Student* b) {
    comparisons++;
    return a->gender - b->gender;
}
// GENDER 내림차순: M < F (1 < 0)
int compare_gender_desc(const Student* a, const Student* b) {
    comparisons++;
    return b->gender - a->gender;
}

// 3.4. 3가지 GRADE의 합 기준 (동점 시 국어, 영어, 수학 순으로 더 큰 사람 우선)
int compare_total_custom(const Student* a, const Student* b) {
    comparisons++;
    if (a->total != b->total) {
        return a->total - b->total; // 오름차순 기준
    }
    // 동점일 경우, 국어 > 영어 > 수학 순으로 더 큰 사람 우선 (내림차순)
    if (a->korean != b->korean) {
        return b->korean - a->korean;
    }
    if (a->english != b->english) {
        return b->english - a->english;
    }
    return b->math - a->math;
}

int compare_total_asc(const Student* a, const Student* b) {
    // a->total < b->total 이면 음수 (a가 앞에 와야 함)
    return compare_total_custom(a, b);
}

int compare_total_desc(const Student* a, const Student* b) {
    // b->total < a->total 이면 음수 (b가 앞에 와야 함 = a가 뒤에 와야 함)
    return compare_total_custom(b, a);
}

// 비교 함수 포인터 타입 정의
typedef int (*Comparator)(const Student*, const Student*);

// --- 4. 헬퍼 함수 (Swap) ---

// 데이터를 교환하고 moves 카운트 증가
void swap_students(Student* a, Student* b) {
    Student temp = *a;
    *a = *b;
    *b = temp;
    moves += 3; // 3번의 할당 (메모리 이동)
}

// --- 5. 9가지 정렬 알고리즘 구현 ---

// 5.1. 버블 정렬 (Bubble Sort)
void bubble_sort(Student arr[], int n, Comparator cmp) {
    for (int i = 0; i < n - 1; i++) {
        int swapped = 0;
        for (int j = 0; j < n - 1 - i; j++) {
            if (cmp(&arr[j], &arr[j + 1]) > 0) {
                swap_students(&arr[j], &arr[j + 1]);
                swapped = 1;
            }
        }
        if (swapped == 0) break;
    }
}

// 5.2. 선택 정렬 (Selection Sort)
void selection_sort(Student arr[], int n, Comparator cmp) {
    for (int i = 0; i < n - 1; i++) {
        int min_idx = i;
        for (int j = i + 1; j < n; j++) {
            if (cmp(&arr[j], &arr[min_idx]) < 0) {
                min_idx = j;
            }
        }
        if (min_idx != i) {
            swap_students(&arr[i], &arr[min_idx]);
        }
    }
}

// 5.3. 삽입 정렬 (Insertion Sort)
void insertion_sort(Student arr[], int n, Comparator cmp) {
    for (int i = 1; i < n; i++) {
        Student key = arr[i];
        moves++; // key를 위한 1번의 이동
        int j = i - 1;

        // 비교: cmp(&arr[j], &key) > 0
        while (j >= 0 && cmp(&arr[j], &key) > 0) {
            arr[j + 1] = arr[j];
            moves++; // 한 칸씩 뒤로 미는 이동
            j = j - 1;
        }
        arr[j + 1] = key;
        moves++; // key를 제 위치에 삽입하는 이동
    }
}

// 5.4. 셸 정렬 (Shell Sort)
void shell_sort(Student arr[], int n, Comparator cmp) {
    for (int gap = n / 2; gap > 0; gap /= 2) {
        for (int i = gap; i < n; i++) {
            Student temp = arr[i];
            moves++;
            int j;
            for (j = i; j >= gap && cmp(&arr[j - gap], &temp) > 0; j -= gap) {
                arr[j] = arr[j - gap];
                moves++;
            }
            arr[j] = temp;
            moves++;
        }
    }
}

// 5.5. 퀵 정렬 (Quick Sort)
int partition(Student arr[], int low, int high, Comparator cmp) {
    Student pivot = arr[high];
    moves++;
    int i = (low - 1);

    for (int j = low; j <= high - 1; j++) {
        if (cmp(&arr[j], &pivot) < 0) {
            i++;
            swap_students(&arr[i], &arr[j]);
        }
    }
    swap_students(&arr[i + 1], &arr[high]);
    return (i + 1);
}

void quick_sort_recursive(Student arr[], int low, int high, Comparator cmp) {
    if (low < high) {
        int mid;

        // 💡 mid에 값 할당
        mid = low + (high - low) / 2;

        // 피벗 선택 전략 개선 (Median-of-Three)
        if (cmp(&arr[mid], &arr[low]) < 0) swap_students(&arr[low], &arr[mid]);
        if (cmp(&arr[high], &arr[low]) < 0) swap_students(&arr[low], &arr[high]);
        if (cmp(&arr[high], &arr[mid]) < 0) swap_students(&arr[mid], &arr[high]);

        int pi = partition(arr, low, high, cmp);
        quick_sort_recursive(arr, low, pi - 1, cmp);
        quick_sort_recursive(arr, pi + 1, high, cmp);
    }
}

void quick_sort(Student arr[], int n, Comparator cmp) {
    quick_sort_recursive(arr, 0, n - 1, cmp);
}

// 5.6. 힙 정렬 (Heap Sort)
void heapify(Student arr[], int n, int i, Comparator cmp) {
    int largest = i;
    int l = 2 * i + 1;
    int r = 2 * i + 2;

    if (l < n && cmp(&arr[l], &arr[largest]) > 0) {
        largest = l;
    }

    if (r < n && cmp(&arr[r], &arr[largest]) > 0) {
        largest = r;
    }

    if (largest != i) {
        swap_students(&arr[i], &arr[largest]);
        heapify(arr, n, largest, cmp);
    }
}

void heap_sort(Student arr[], int n, Comparator cmp) {
    for (int i = n / 2 - 1; i >= 0; i--) {
        heapify(arr, n, i, cmp);
    }

    for (int i = n - 1; i > 0; i--) {
        swap_students(&arr[0], &arr[i]);
        heapify(arr, i, 0, cmp);
    }
}

// 5.7. 병합 정렬 (Merge Sort)
void merge(Student arr[], int l, int m, int r, Comparator cmp) {
    int i, j, k;
    int n1 = m - l + 1;
    int n2 = r - m;

    Student* L = (Student*)malloc(n1 * sizeof(Student));
    Student* R = (Student*)malloc(n2 * sizeof(Student));

    if (!L || !R) {
        if (L) free(L);
        if (R) free(R);
        return; // 메모리 할당 실패 처리
    }

    for (i = 0; i < n1; i++) {
        L[i] = arr[l + i];
        moves++; // 임시 배열로의 이동
    }
    for (j = 0; j < n2; j++) {
        R[j] = arr[m + 1 + j];
        moves++; // 임시 배열로의 이동
    }

    i = 0;
    j = 0;
    k = l;
    while (i < n1 && j < n2) {
        if (cmp(&L[i], &R[j]) <= 0) {
            arr[k] = L[i];
            i++;
        }
        else {
            arr[k] = R[j];
            j++;
        }
        moves++; // 본 배열로의 이동
        k++;
    }

    while (i < n1) {
        arr[k] = L[i];
        moves++;
        i++;
        k++;
    }

    while (j < n2) {
        arr[k] = R[j];
        moves++;
        j++;
        k++;
    }

    free(L);
    free(R);
}

void merge_sort_recursive(Student arr[], int l, int r, Comparator cmp) {
    if (l < r) {
        int m = l + (r - l) / 2;
        merge_sort_recursive(arr, l, m, cmp);
        merge_sort_recursive(arr, m + 1, r, cmp);
        merge(arr, l, m, r, cmp);
    }
}

void merge_sort(Student arr[], int n, Comparator cmp) {
    merge_sort_recursive(arr, 0, n - 1, cmp);
}

// 5.8. 기수 정렬 (Radix Sort - ID 기준 오름차순에만 적용 가능)
// 기수 정렬은 Student 구조체 전체를 비교하는 Comparator와 맞지 않으므로,
// ID 필드에 특화된 별도의 함수로 구현합니다.
void count_sort_id(Student arr[], int n, int exp) {
    Student* output = (Student*)malloc(n * sizeof(Student));
    int i;
    int count[10] = { 0 };

    if (!output) return;

    // 현재 자릿수의 빈도 계산
    for (i = 0; i < n; i++) {
        count[(arr[i].id / exp) % 10]++;
    }

    // 누적 빈도 계산
    for (i = 1; i < 10; i++) {
        count[i] += count[i - 1];
    }

    // 출력 배열 구축
    for (i = n - 1; i >= 0; i--) {
        output[count[(arr[i].id / exp) % 10] - 1] = arr[i];
        moves++; // 임시 배열로의 이동
        count[(arr[i].id / exp) % 10]--;
    }

    // arr로 복사
    for (i = 0; i < n; i++) {
        arr[i] = output[i];
        moves++; // 본 배열로의 이동
    }

    free(output);
    // 참고: Radix Sort는 비교 기반이 아니므로 comparisons는 0으로 유지됨
}

void radix_sort(Student arr[], int n, Comparator cmp) {
    // ID 필드의 최댓값을 찾아야 함.
    int max_id = arr[0].id;
    for (int i = 1; i < n; i++) {
        if (arr[i].id > max_id) {
            max_id = arr[i].id;
        }
    }

    // 자릿수별로 Count Sort 수행
    for (int exp = 1; max_id / exp > 0; exp *= 10) {
        count_sort_id(arr, n, exp);
    }
}

// 5.9. 트리 정렬 (Tree Sort)
// 이진 탐색 트리 (Binary Search Tree)를 사용하여 구현합니다.
// 요구사항에 따라 힙/트리 정렬은 중복 데이터가 있는 경우 수행하지 않지만,
// ID를 primary key로 사용하면 중복이 없으므로, ID 오름차순 정렬만 구현합니다.

typedef struct node {
    Student data;
    struct node* left, * right;
} Node;

Node* new_node(Student data) {
    Node* temp = (Node*)malloc(sizeof(Node));
    if (temp) {
        temp->data = data;
        moves++;
        temp->left = temp->right = NULL;
    }
    return temp;
}

Node* insert_node(Node* node, Student data) {
    if (node == NULL) return new_node(data);

    // ID 오름차순 기준으로만 정렬 (중복 불가)
    if (data.id < node->data.id) {
        comparisons++;
        node->left = insert_node(node->left, data);
    }
    else if (data.id > node->data.id) {
        comparisons++;
        node->right = insert_node(node->right, data);
    }
    // 같은 ID는 무시 (중복 데이터 처리 생략, 실제 데이터셋에 중복 ID가 없다고 가정)
    return node;
}

void inorder_traversal(Node* root, Student arr[], int* index) {
    if (root != NULL) {
        inorder_traversal(root->left, arr, index);
        arr[(*index)++] = root->data;
        moves++; // 배열로의 복사 (이동)
        inorder_traversal(root->right, arr, index);
    }
}

void delete_tree(Node* node) {
    if (node == NULL) return;
    delete_tree(node->left);
    delete_tree(node->right);
    free(node);
}

void tree_sort(Student arr[], int n, Comparator cmp) {
    // ID 기준으로만 정렬 가능 (ID는 중복이 없다고 가정)
    if (cmp != compare_id_asc) {
        fprintf(stderr, "Warning: Tree Sort is only implemented for ID ascending in this code.\n");
        return;
    }

    Node* root = NULL;
    for (int i = 0; i < n; i++) {
        root = insert_node(root, arr[i]);
    }

    int index = 0;
    inorder_traversal(root, arr, &index);

    delete_tree(root);
}


// --- 6. 실행 및 측정 로직 ---

typedef void (*SortFunction)(Student[], int, Comparator);

void run_and_measure(const char* algo_name, SortFunction sort_func, Student original_data[], int n,
    const char* criterion_name, Comparator cmp, int skip_dup_check) {

    // 중복 데이터 체크: ID, NAME, GENDER 필드를 제외한 중복 데이터 존재 여부
    // 여기서는 간단히 ID가 아닌 기준으로 정렬할 때만 중복 가능성이 있다고 가정합니다.
    if (skip_dup_check && (cmp != compare_id_asc && cmp != compare_id_desc)) {
        printf("| %-18s | %-45s | (SKIP: 중복 데이터 가능성으로 인해 건너뜀) \n", algo_name, criterion_name);
        return;
    }

    // 1000회 반복 측정
    long long total_comparisons = 0;
    long long total_moves = 0;
    clock_t start_time, end_time;
    double total_time = 0;

    for (int i = 0; i < REPETITIONS; i++) {
        // 정렬할 배열 복사 (매 반복마다 원본 사용)
        Student* working_records = (Student*)malloc(n * sizeof(Student));
        if (!working_records) {
            perror("작업_레코드에 대한 메모리 할당 실패");
            return;
        }
        memcpy(working_records, original_data, sizeof(Student) * n);

        // 측정 변수 초기화
        comparisons = 0;
        moves = 0;

        // 정렬 시작 시간 측정
        start_time = clock();

        // 정렬 함수 실행
        if (strcmp(algo_name, "Radix Sort") == 0) {
            // Radix Sort는 ID 오름차순 정렬에 특화되어 있으므로, cmp 인자를 무시합니다.
            radix_sort(working_records, n, NULL);
        }
        else {
            sort_func(working_records, n, cmp);
        }

        // 정렬 종료 시간 측정
        end_time = clock();

        // 결과 누적
        total_comparisons += comparisons;
        total_moves += moves;
        total_time += (double)(end_time - start_time) / CLOCKS_PER_SEC;

        free(working_records);
    }

    // 평균 계산
    double avg_comparisons = (double)total_comparisons / REPETITIONS;
    double avg_moves = (double)total_moves / REPETITIONS;
    double avg_time_ms = (total_time / REPETITIONS) * 1000.0;

    // 결과 출력
    printf("| %-18s | %-45s | %18.2f | %18.2f | %15.3fms |\n",
        algo_name, criterion_name, avg_comparisons, avg_moves, avg_time_ms);
}

// --- 7. 메인 함수 ---

int main() {
    // 1. 데이터 로드 및 전처리
    int record_count = 0;
    Student* original_records = load_students(FILENAME, &record_count);

    if (record_count == 0) {
        printf("오류: 로드된 데이터가 없습니다 %s.\n", FILENAME);
        return 1;
    }
    printf("데이터가 성공적으로 로드되었습니다. 총 기록: %d\n\n", record_count);

    // 2. 정렬 알고리즘 및 기준 정의
    typedef struct {
        const char* name;
        SortFunction func;
    } SortAlgo;

    typedef struct {
        const char* name;
        Comparator cmp;
        int is_stable;
        int is_radix_valid; // Radix Sort 사용 가능 여부
    } SortCriterion;

    SortAlgo algorithms[] = {
        {"Bubble Sort", bubble_sort},
        {"Selection Sort", selection_sort},
        {"Insertion Sort", insertion_sort},
        {"Shell Sort", shell_sort},
        {"Quick Sort", quick_sort},
        {"Heap Sort", heap_sort},
        {"Merge Sort", merge_sort},
        {"Radix Sort", radix_sort},
        {"Tree Sort", tree_sort}
    };

    SortCriterion criteria[] = {
        {"ID Ascending", compare_id_asc, 0, 1},
        {"ID Descending", compare_id_desc, 0, 0},
        {"NAME Ascending", compare_name_asc, 0, 0},
        {"NAME Descending", compare_name_desc, 0, 0},
        {"GENDER Ascending (Stable)", compare_gender_asc, 1, 0},
        {"GENDER Descending (Stable)", compare_gender_desc, 1, 0},
        {"Total Grade Ascending (Custom Tie)", compare_total_asc, 0, 0},
        {"Total Grade Descending (Custom Tie)", compare_total_desc, 0, 0}
    };

    // 3. 결과 출력 헤더
    printf("==============================================================================================================================\n");
    printf("| %-18s | %-45s | %-18s | %-18s | %-15s |\n",
        "알고리즘", "기준", "평균 비교", "평균 이동(메모리)", "평균 시간(ms)");
    printf("==============================================================================================================================\n");

    // 4. 모든 조합 실행 및 측정
    for (size_t i = 0; i < sizeof(algorithms) / sizeof(algorithms[0]); i++) {
        for (size_t j = 0; j < sizeof(criteria) / sizeof(criteria[0]); j++) {

            // --- 정렬 기준 예외 처리 ---

            // 1. GENDER 정렬은 Stable 정렬(Insertion, Merge)만 사용
            if (criteria[j].is_stable) {
                if (strcmp(algorithms[i].name, "Insertion Sort") != 0 &&
                    strcmp(algorithms[i].name, "Merge Sort") != 0) {
                    continue; // Stable 정렬이 아니면 건너뜀
                }
            }

            // 2. Radix Sort는 ID Ascending에만 사용
            if (strcmp(algorithms[i].name, "Radix Sort") == 0) {
                if (!criteria[j].is_radix_valid) continue;
            }

            // 3. 힙/트리 정렬은 중복 데이터가 있는 경우 제외
            // ID를 제외한 다른 기준으로 정렬할 때 중복 데이터가 있다고 가정하고 제외합니다.
            int skip_for_duplicate = (strcmp(algorithms[i].name, "Heap Sort") == 0 ||
                strcmp(algorithms[i].name, "Tree Sort") == 0) &&
                !criteria[j].is_radix_valid; // ID 정렬이 아니면 스킵

            run_and_measure(algorithms[i].name, algorithms[i].func,
                original_records, record_count,
                criteria[j].name, criteria[j].cmp, skip_for_duplicate);
        }
    }

    printf("==============================================================================================================================\n");

    // 메모리 해제
    free(original_records);

    return 0;
}
