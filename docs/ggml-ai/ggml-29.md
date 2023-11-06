# GGML源码解析 29

# `tests/test-xpos.c`

0.0001, 2.8705, 1.8665, 2.1479,
0.2083, -2.8486, -1.7103, 1.9251,
2.5158, -1.3671, -2.3146, 1.9057,
2.5158, 1.3816, 1.8862, 2.1273,
0.1972, 2.8705, 1.8665, 2.1479,
-2.3146, 1.7211, -1.7103, 1.9441,
-2.2789, -1.7103, -1.7103, 1.9441,
2.0644, 1.9251, 2.0856, 2.1684,
2.5158, -1.3671, -2.3146, 1.8862,
2.5158, -1.3671, -2.3146, 1.9251,
2.0644, 1.9251, 2.0856, 2.1479,
2.5158, -1.3671, -2.3146, 1.9057,
2.5158, -1.3671, -2.3146, 1.8862,
2.0644, 1.9251, 2.0856, 2.1684,
2.5158, -1.3671, -2.3146, 1.9441,


```cpp
#include "ggml/ggml.h"

#include <math.h>
#include <stdio.h>
#include <stdlib.h>

bool is_close(float a, float b, float epsilon) {
    return fabs(a - b) < epsilon;
}

int main(int argc, char ** argv) {
    const int n_threads = 1;
    const int n_embd_head = 4; // aka head_dim
    const int n_head = 1;
    const int N = 8;

    struct ggml_init_params params = {
        .mem_size   = 16*1024*1024,
        .mem_buffer = NULL,
    };

    // memory allocation happens here
    struct ggml_context * ctx = ggml_init(params);

    struct ggml_tensor * Q = ggml_new_tensor_3d(ctx, GGML_TYPE_F32, n_embd_head, n_head, N);
    struct ggml_tensor * K = ggml_new_tensor_3d(ctx, GGML_TYPE_F32, n_embd_head, n_head, N);

    for (int i = 0; i < ggml_nelements(Q); i++) {
        ((float*) Q->data)[i] = 2.0f;
        ((float*) K->data)[i] = 2.0f;
    }

    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, N);
    int * data = (int *) KQ_pos->data;
    for (int i = 0; i < N; ++i) {
        data[i] = 1 + i;
    }

    struct ggml_tensor * Qx = ggml_rope_xpos_inplace(ctx, Q, KQ_pos, n_embd_head, 512.0f, false);
    struct ggml_tensor * Kx = ggml_rope_xpos_inplace(ctx, K, KQ_pos, n_embd_head, 512.0f, true);

    struct ggml_cgraph * gf = ggml_new_graph(ctx);
    ggml_build_forward_expand(gf, Qx);
    ggml_build_forward_expand(gf, Kx);
    ggml_graph_compute_with_ctx(ctx, gf, n_threads);

	// expected output for Qx:
    // -0.6009  2.7568  1.9782  2.0182
    // -2.6379  0.9815  1.9562  2.0361
    // -2.2457 -1.6853  1.9341  2.0538
    //  0.2043 -2.7934  1.9118  2.0712
    //  2.4550 -1.3341  1.8894  2.0884
    //  2.4430  1.3417  1.8668  2.1054
    //  0.1905  2.7739  1.8440  2.1221
    // -2.2257  1.6550  1.8212  2.1386

    for (int i = 0; i < ggml_nelements(Q); i++) {
        if (((float*) Qx->data)[i] > 0) printf(" ");
        printf("%.4f ", ((float*) Qx->data)[i]);
        if ((i+1) % n_embd_head == 0) printf("\n");
    }
    printf("\n");

    GGML_ASSERT(is_close(((float*) Qx->data)[7 * n_embd_head + 0], -2.2257f, 0.0001f));
    GGML_ASSERT(is_close(((float*) Qx->data)[7 * n_embd_head + 1],  1.6550f, 0.0001f));
    GGML_ASSERT(is_close(((float*) Qx->data)[7 * n_embd_head + 2],  1.8212f, 0.0001f));
    GGML_ASSERT(is_close(((float*) Qx->data)[7 * n_embd_head + 3],  2.1386f, 0.0001f));

    // expected output for Kx:
	// -0.6038  2.7703  1.9816  2.0216
    // -2.6639  0.9911  1.9630  2.0431
    // -2.2789 -1.7103  1.9441  2.0644
    //  0.2083 -2.8486  1.9251  2.0856
    //  2.5158 -1.3671  1.9057  2.1065
    //  2.5158  1.3816  1.8862  2.1273
    //  0.1972  2.8705  1.8665  2.1479
    // -2.3146  1.7211  1.8465  2.1684

    for (int i = 0; i < ggml_nelements(K); i++) {
        if (((float*) Kx->data)[i] > 0) printf(" ");
        printf("%.4f ", ((float*) Kx->data)[i]);
        if ((i+1) % n_embd_head == 0) printf("\n");
    }
    printf("\n");

    GGML_ASSERT(is_close(((float*) Kx->data)[7 * n_embd_head + 0], -2.3146f, 0.0001f));
    GGML_ASSERT(is_close(((float*) Kx->data)[7 * n_embd_head + 1],  1.7211f, 0.0001f));
    GGML_ASSERT(is_close(((float*) Kx->data)[7 * n_embd_head + 2],  1.8465f, 0.0001f));
    GGML_ASSERT(is_close(((float*) Kx->data)[7 * n_embd_head + 3],  2.1684f, 0.0001f));

    ggml_free(ctx);

    return 0;
}

```

# `tests/test0.c`

This is a C function that performs a 2D matrix multiplication using the Gl深度-to-one layer. The input matrices are represented as 2D tensors of the form (10,) and (10, 20), and the output matrix is represented as a 3D tensor of the form (10, 20, 30). The function uses the `ggml_new_tensor_2d()` and `ggml_new_tensor_3d()` functions to create the input and output tensors, respectively. The `GGML_ASSERT()` macro is used to check the dimension and element size of the tensors. The function prints the input and output tensors using the `ggml_print_objects()` function, and then frees the memory allocated by the function using the `ggml_free()` function.


```cpp
#include "ggml/ggml.h"

#include <stdio.h>
#include <stdlib.h>

int main(int argc, const char ** argv) {
    struct ggml_init_params params = {
        .mem_size   = 128*1024*1024,
        .mem_buffer = NULL,
        .no_alloc   = false,
    };

    struct ggml_context * ctx0 = ggml_init(params);

    struct ggml_tensor * t1 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 10);
    struct ggml_tensor * t2 = ggml_new_tensor_2d(ctx0, GGML_TYPE_I16, 10, 20);
    struct ggml_tensor * t3 = ggml_new_tensor_3d(ctx0, GGML_TYPE_I32, 10, 20, 30);

    GGML_ASSERT(t1->n_dims == 1);
    GGML_ASSERT(t1->ne[0]  == 10);
    GGML_ASSERT(t1->nb[1]  == 10*sizeof(float));

    GGML_ASSERT(t2->n_dims == 2);
    GGML_ASSERT(t2->ne[0]  == 10);
    GGML_ASSERT(t2->ne[1]  == 20);
    GGML_ASSERT(t2->nb[1]  == 10*sizeof(int16_t));
    GGML_ASSERT(t2->nb[2]  == 10*20*sizeof(int16_t));

    GGML_ASSERT(t3->n_dims == 3);
    GGML_ASSERT(t3->ne[0]  == 10);
    GGML_ASSERT(t3->ne[1]  == 20);
    GGML_ASSERT(t3->ne[2]  == 30);
    GGML_ASSERT(t3->nb[1]  == 10*sizeof(int32_t));
    GGML_ASSERT(t3->nb[2]  == 10*20*sizeof(int32_t));
    GGML_ASSERT(t3->nb[3]  == 10*20*30*sizeof(int32_t));

    ggml_print_objects(ctx0);

    ggml_free(ctx0);

    return 0;
}

```

# `tests/test1.c`

This is a C program that uses GGML (Graph Plotting Markup Language) to draw a two-dimensional grid of 32 x 32 points, and optionally calculate the gradient of the红色 point on each grid point.

The program takes two arguments:

* The first argument is a two-dimensional grid of points, represented by the GGML variable `x1` and `y1`.
* The second argument is a pointer to a structure that implements the `IWorker` interface and contains the gradient of the red point at each grid point.

The program first sets the red point as the first point in the grid by setting `x1[0] = 1` and `y1[0] = 1`. Then, it initializes the `IWorker` instance with a pointer to `ggml_get_f32_1d(x1[0], 0)` to calculate the gradient at the red point.

Next, the program loops through the grid and calculates the gradient of the red point using the `IWorker` instance. Finally, it prints the result to the console.

The program uses the `ggml_graph_dump_dot` function to print the graph to the console, and also uses the `ggml_graph_dump_dot` function with the `gb` graph object to store the graph in memory for later dumping.


```cpp
#include "ggml/ggml.h"

#include <stdio.h>
#include <stdlib.h>

int main(int argc, const char ** argv) {
    const int n_threads = 2;

    struct ggml_init_params params = {
        .mem_size   = 128*1024*1024,
        .mem_buffer = NULL,
        .no_alloc   = false,
    };

    struct ggml_context * ctx0 = ggml_init(params);

    {
        struct ggml_tensor * x = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);

        ggml_set_param(ctx0, x);

        struct ggml_tensor * a = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
        struct ggml_tensor * b = ggml_mul(ctx0, x, x);
        struct ggml_tensor * f = ggml_mul(ctx0, b, a);

        // a*x^2
        // 2*a*x

        ggml_print_objects(ctx0);

        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
        ggml_build_forward_expand(gf, f);
        struct ggml_cgraph * gb = ggml_graph_dup(ctx0, gf);
        ggml_build_backward_expand(ctx0, gf, gb, false);

        ggml_set_f32(x, 2.0f);
        ggml_set_f32(a, 3.0f);

        ggml_graph_reset(gf);
        ggml_set_f32(f->grad, 1.0f);

        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        printf("f     = %f\n", ggml_get_f32_1d(f, 0));
        printf("df/dx = %f\n", ggml_get_f32_1d(x->grad, 0));

        GGML_ASSERT(ggml_get_f32_1d(f, 0)       == 12.0f);
        GGML_ASSERT(ggml_get_f32_1d(x->grad, 0) == 12.0f);

        ggml_set_f32(x, 3.0f);

        ggml_graph_reset(gf);
        ggml_set_f32(f->grad, 1.0f);

        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        printf("f     = %f\n", ggml_get_f32_1d(f, 0));
        printf("df/dx = %f\n", ggml_get_f32_1d(x->grad, 0));

        GGML_ASSERT(ggml_get_f32_1d(f, 0)       == 27.0f);
        GGML_ASSERT(ggml_get_f32_1d(x->grad, 0) == 18.0f);

        ggml_graph_dump_dot(gf, NULL, "test1-1-forward.dot");
        ggml_graph_dump_dot(gb, gf,   "test1-1-backward.dot");
    }

    ///////////////////////////////////////////////////////////////

    {
        struct ggml_tensor * x1 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
        struct ggml_tensor * x2 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
        struct ggml_tensor * x3 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);

        ggml_set_f32(x1, 3.0f);
        ggml_set_f32(x2, 1.0f);
        ggml_set_f32(x3, 0.0f);

        ggml_set_param(ctx0, x1);
        ggml_set_param(ctx0, x2);

        struct ggml_tensor * y = ggml_add(ctx0, ggml_mul(ctx0, x1, x1), ggml_mul(ctx0, x1, x2));

        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
        ggml_build_forward_expand(gf, y);
        struct ggml_cgraph * gb = ggml_graph_dup(ctx0, gf);
        ggml_build_backward_expand(ctx0, gf, gb, false);

        ggml_graph_reset(gf);
        ggml_set_f32(y->grad, 1.0f);

        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        printf("y      = %f\n", ggml_get_f32_1d(y, 0));
        printf("df/dx1 = %f\n", ggml_get_f32_1d(x1->grad, 0));
        printf("df/dx2 = %f\n", ggml_get_f32_1d(x2->grad, 0));

        GGML_ASSERT(ggml_get_f32_1d(y, 0)        == 12.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) == 7.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) == 3.0f);

        struct ggml_tensor * g1 = x1->grad;
        struct ggml_tensor * g2 = x2->grad;

        struct ggml_cgraph * gbb = ggml_graph_dup(ctx0, gb);

        ggml_build_backward_expand(ctx0, gb, gbb, true);

        ggml_graph_reset(gb);
        ggml_set_f32(g1->grad, 1.0f);
        ggml_set_f32(g2->grad, 1.0f);

        ggml_graph_compute_with_ctx(ctx0, gbb, n_threads);

        printf("H * [1, 1] = [ %f %f ]\n", ggml_get_f32_1d(x1->grad, 0), ggml_get_f32_1d(x2->grad, 0));

        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) == 3.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) == 1.0f);

        ggml_graph_dump_dot(gf, NULL, "test1-2-forward.dot");
        ggml_graph_dump_dot(gb, gf,   "test1-2-backward.dot");
    }

    ///////////////////////////////////////////////////////////////

    {
        struct ggml_tensor * x1 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
        struct ggml_tensor * x2 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);

        ggml_set_param(ctx0, x1);
        ggml_set_param(ctx0, x2);

        struct ggml_tensor * y = ggml_mul(ctx0, ggml_add(ctx0, ggml_mul(ctx0, x1, x1), ggml_mul(ctx0, x1, x2)), x1);

        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
        ggml_build_forward_expand(gf, y);
        struct ggml_cgraph * gb = ggml_graph_dup(ctx0, gf);
        ggml_build_backward_expand(ctx0, gf, gb, false);

        ggml_set_f32(x1, 3.0f);
        ggml_set_f32(x2, 4.0f);

        ggml_graph_reset(gf);
        ggml_set_f32(y->grad, 1.0f);

        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        printf("y      = %f\n", ggml_get_f32_1d(y, 0));
        printf("df/dx1 = %f\n", ggml_get_f32_1d(x1->grad, 0));
        printf("df/dx2 = %f\n", ggml_get_f32_1d(x2->grad, 0));

        GGML_ASSERT(ggml_get_f32_1d(y, 0)        == 63.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) == 51.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) == 9.0f);

        ggml_graph_dump_dot(gf, NULL, "test1-3-forward.dot");
        ggml_graph_dump_dot(gb, gf,   "test1-3-backward.dot");
    }

    ///////////////////////////////////////////////////////////////

    {
        struct ggml_tensor * x1 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
        struct ggml_tensor * x2 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
        struct ggml_tensor * x3 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);

        ggml_set_param(ctx0, x1);
        ggml_set_param(ctx0, x2);
        ggml_set_param(ctx0, x3);

        struct ggml_tensor * y = ggml_mul(ctx0, ggml_mul(ctx0, ggml_mul(ctx0, x1, x1), ggml_mul(ctx0, x2, x2)), x3);

        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
        ggml_build_forward_expand(gf, y);
        struct ggml_cgraph * gb = ggml_graph_dup(ctx0, gf);
        ggml_build_backward_expand(ctx0, gf, gb, false);

        ggml_set_f32(x1, 1.0f);
        ggml_set_f32(x2, 2.0f);
        ggml_set_f32(x3, 3.0f);

        ggml_graph_reset(gf);
        ggml_set_f32(y->grad, 1.0f);

        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        printf("y      = %f\n", ggml_get_f32_1d(y, 0));
        printf("df/dx1 = %f\n", ggml_get_f32_1d(x1->grad, 0));
        printf("df/dx2 = %f\n", ggml_get_f32_1d(x2->grad, 0));
        printf("df/dx3 = %f\n", ggml_get_f32_1d(x3->grad, 0));

        GGML_ASSERT(ggml_get_f32_1d(y, 0)        == 12.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) == 24.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) == 12.0f);
        GGML_ASSERT(ggml_get_f32_1d(x3->grad, 0) == 4.0f);

        struct ggml_tensor * g1 = x1->grad;
        struct ggml_tensor * g2 = x2->grad;
        struct ggml_tensor * g3 = x3->grad;

        struct ggml_cgraph * gbb = ggml_graph_dup(ctx0, gb);

        ggml_build_backward_expand(ctx0, gb, gbb, true);

        ggml_graph_reset(gb);
        ggml_set_f32(g1->grad, 1.0f);
        ggml_set_f32(g2->grad, 1.0f);
        ggml_set_f32(g3->grad, 1.0f);

        ggml_graph_compute_with_ctx(ctx0, gbb, n_threads);

        printf("H * [1, 1, 1] = [ %f %f %f ]\n",
                ggml_get_f32_1d(x1->grad, 0),
                ggml_get_f32_1d(x2->grad, 0),
                ggml_get_f32_1d(x3->grad, 0));

        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) == 56.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) == 34.0f);
        GGML_ASSERT(ggml_get_f32_1d(x3->grad, 0) == 12.0f);

        ggml_graph_dump_dot(gf, NULL, "test1-4-forward.dot");
        ggml_graph_dump_dot(gb, gf,   "test1-4-backward.dot");
    }

    ///////////////////////////////////////////////////////////////

    {
        struct ggml_tensor * x1 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 3);
        struct ggml_tensor * x2 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 3);

        ggml_set_param(ctx0, x1);
        ggml_set_param(ctx0, x2);

        struct ggml_tensor * y = ggml_sum(ctx0, ggml_mul(ctx0, x1, x2));

        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
        ggml_build_forward_expand(gf, y);
        struct ggml_cgraph * gb = ggml_graph_dup(ctx0, gf);
        ggml_build_backward_expand(ctx0, gf, gb, false);

        ggml_set_f32(x1, 3.0f);
        ggml_set_f32(x2, 5.0f);

        ggml_graph_reset(gf);
        ggml_set_f32(y->grad, 1.0f);

        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        printf("y      = %f\n", ggml_get_f32_1d(y, 0));
        printf("df/dx1 = %f %f %f\n",
                ggml_get_f32_1d(x1->grad, 0),
                ggml_get_f32_1d(x1->grad, 1),
                ggml_get_f32_1d(x1->grad, 2));
        printf("df/dx2 = %f %f %f\n",
                ggml_get_f32_1d(x2->grad, 0),
                ggml_get_f32_1d(x2->grad, 1),
                ggml_get_f32_1d(x2->grad, 2));

        GGML_ASSERT(ggml_get_f32_1d(y, 0)        == 45.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) == 5.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) == 3.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 1) == 5.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 1) == 3.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 2) == 5.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 2) == 3.0f);

        ggml_graph_dump_dot(gf, NULL, "test1-5-forward.dot");
        ggml_graph_dump_dot(gb, gf,   "test1-5-backward.dot");
    }

    ///////////////////////////////////////////////////////////////

    {
        struct ggml_tensor * x1 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 3);
        struct ggml_tensor * x2 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 3);

        ggml_set_param(ctx0, x1);
        ggml_set_param(ctx0, x2);

        struct ggml_tensor * y =
            ggml_sum(ctx0,
                    ggml_add(ctx0,
                        ggml_mul(ctx0, x1, x2),
                        ggml_mul(ctx0,
                            ggml_repeat(ctx0, ggml_new_f32(ctx0, -2.0f), x1),
                            ggml_mul(ctx0, x1, x1)
                            )
                        )
                    );

        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
        ggml_build_forward_expand(gf, y);
        struct ggml_cgraph * gb = ggml_graph_dup(ctx0, gf);
        ggml_build_backward_expand(ctx0, gf, gb, false);

        ggml_set_f32(x1, 3.0f);
        ggml_set_f32(x2, 5.0f);

        ggml_graph_reset(gf);
        ggml_set_f32(y->grad, 1.0f);

        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        printf("y      = %f\n", ggml_get_f32_1d(y, 0));
        printf("df/dx1 = %f %f %f\n",
                ggml_get_f32_1d(x1->grad, 0),
                ggml_get_f32_1d(x1->grad, 1),
                ggml_get_f32_1d(x1->grad, 2));
        printf("df/dx2 = %f %f %f\n",
                ggml_get_f32_1d(x2->grad, 0),
                ggml_get_f32_1d(x2->grad, 1),
                ggml_get_f32_1d(x2->grad, 2));

        GGML_ASSERT(ggml_get_f32_1d(y, 0)              == -9.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) == -7.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 1) == -7.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 2) == -7.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) ==  3.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 1) ==  3.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 2) ==  3.0f);

        ggml_graph_dump_dot(gf, NULL, "test1-6-forward.dot");
        ggml_graph_dump_dot(gb, gf,   "test1-6-backward.dot");
    }

    ///////////////////////////////////////////////////////////////

    {
        struct ggml_tensor * x1 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 3);
        struct ggml_tensor * x2 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 3);

        ggml_set_param(ctx0, x1);
        ggml_set_param(ctx0, x2);

        struct ggml_tensor * y =
            ggml_sum(ctx0,
                    ggml_sub(ctx0,
                        ggml_mul(ctx0, x1, x2),
                        ggml_mul(ctx0,
                            ggml_mul(ctx0, x1, x1),
                            ggml_repeat(ctx0, ggml_new_f32(ctx0, -2.0f), x1)
                            )
                        )
                    );

        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
        ggml_build_forward_expand(gf, y);
        struct ggml_cgraph * gb = ggml_graph_dup(ctx0, gf);
        ggml_build_backward_expand(ctx0, gf, gb, false);

        ggml_set_f32(x1, 3.0f);
        ggml_set_f32(x2, 5.0f);

        ggml_graph_reset(gf);
        ggml_set_f32(y->grad, 1.0f);

        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        printf("y      = %f\n", ggml_get_f32_1d(y, 0));
        printf("df/dx1 = %f %f %f\n",
                ggml_get_f32_1d(x1->grad, 0),
                ggml_get_f32_1d(x1->grad, 1),
                ggml_get_f32_1d(x1->grad, 2));
        printf("df/dx2 = %f %f %f\n",
                ggml_get_f32_1d(x2->grad, 0),
                ggml_get_f32_1d(x2->grad, 1),
                ggml_get_f32_1d(x2->grad, 2));

        GGML_ASSERT(ggml_get_f32_1d(y, 0)        == 99.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) == 17.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 1) == 17.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 2) == 17.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) ==  3.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 1) ==  3.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 2) ==  3.0f);

        ggml_graph_dump_dot(gf, NULL, "test1-7-forward.dot");
        ggml_graph_dump_dot(gb, gf,   "test1-7-backward.dot");
    }

    ///////////////////////////////////////////////////////////////

    {
        struct ggml_tensor * x1 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 3);
        struct ggml_tensor * x2 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 3);

        ggml_set_param(ctx0, x1);
        ggml_set_param(ctx0, x2);

        struct ggml_tensor * y =
            ggml_abs(ctx0,
                    ggml_sub(ctx0, x1, x2)
                    );

        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
        ggml_build_forward_expand(gf, y);
        struct ggml_cgraph * gb = ggml_graph_dup(ctx0, gf);
        ggml_build_backward_expand(ctx0, gf, gb, false);

        ggml_set_f32(x1, 3.0f);
        ggml_set_f32(x2, 5.0f);

        ggml_graph_reset(gf);
        ggml_set_f32(y->grad, 1.0f);

        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        printf("y      = %f\n", ggml_get_f32_1d(y, 0));
        printf("df/dx1 = %f %f %f\n",
                ggml_get_f32_1d(x1->grad, 0),
                ggml_get_f32_1d(x1->grad, 1),
                ggml_get_f32_1d(x1->grad, 2));
        printf("df/dx2 = %f %f %f\n",
                ggml_get_f32_1d(x2->grad, 0),
                ggml_get_f32_1d(x2->grad, 1),
                ggml_get_f32_1d(x2->grad, 2));

        GGML_ASSERT(ggml_get_f32_1d(y, 0)        ==  2.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) == -1.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 1) == -1.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 2) == -1.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) ==  1.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 1) ==  1.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 2) ==  1.0f);

        ggml_set_f32(x1, 7.0f);
        ggml_set_f32(x2, 5.0f);

        ggml_graph_reset(gf);
        ggml_set_f32(y->grad, 1.0f);

        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        printf("y      = %f\n", ggml_get_f32_1d(y, 0));
        printf("df/dx1 = %f %f %f\n",
                ggml_get_f32_1d(x1->grad, 0),
                ggml_get_f32_1d(x1->grad, 1),
                ggml_get_f32_1d(x1->grad, 2));
        printf("df/dx2 = %f %f %f\n",
                ggml_get_f32_1d(x2->grad, 0),
                ggml_get_f32_1d(x2->grad, 1),
                ggml_get_f32_1d(x2->grad, 2));

        GGML_ASSERT(ggml_get_f32_1d(y, 0)        ==  2.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) ==  1.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 1) ==  1.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 2) ==  1.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) == -1.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 1) == -1.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 2) == -1.0f);

        ggml_graph_dump_dot(gf, NULL, "test1-8-forward.dot");
        ggml_graph_dump_dot(gb, gf,   "test1-8-backward.dot");
    }

    ggml_free(ctx0);

    return 0;
}

```

# `tests/test2.c`

这段代码定义了一个名为`_CRT_SECURE_NO_DEPRECATE`的宏，它在`#define`语句中声明了一个预处理指令，用于将`unsafe`警告disabled。这个宏的含义是：如果定义和使用`_CRT_SECURE_NO_DEPRECATE`宏的上下文是在Windows环境下，那么编译器会禁止对`unsafe`警告进行警告，从而避免在 warning 级别上出现一些过时的、不安全的代码。

接下来，该代码引入了`ggml/ggml.h`头文件，该头文件可能是用于在`ggml`库中使用某些标准函数或类的头文件。

然后，该代码包括了一系列数学和标准输入输出库的包含头文件，如`<math.h>`，`<stdio.h>`和`<stdlib.h>`，这些头文件中定义了一些常用的数学函数和输入输出库。

接着，该代码定义了一个名为`is_close`的函数，该函数接受两个浮点数参数`a`和`b`，以及一个浮点数参数`epsilon`。该函数用于检查两个参数之间的差的绝对值是否小于给定的epsilon。

最后，该代码包括了一些预处理指令，如`#if defined(_MSC_VER)`和`#pragma warning(disable: 4244 4267)`。前者用于检查定义该代码的上下文是否使用了_MSC_VER，如果是，则会禁用4244和4267警告。后者用于禁用一些警告，以防止在代码中使用已经过时的或者不安全的C语言特性。


```cpp
#define _CRT_SECURE_NO_DEPRECATE // Disables ridiculous "unsafe" warnigns on Windows
#include "ggml/ggml.h"

#include <math.h>
#include <stdio.h>
#include <stdlib.h>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

bool is_close(float a, float b, float epsilon) {
    return fabs(a - b) < epsilon;
}

```

This function appears to calculate the value of the function f(t0, t1) = ((t0 + 2*t1 - 7) ^ 2 + (2*t0 + t1 - 5) ^ 2) / (2^ 8).0f), where t0 and t1 are given values ranging from 0 to 32, and the value of the function is represented by the smaller value of the two operands. The function uses some custom logic to calculate the value of the function, which is then compared to a close value to ensure the function returns a reasonable result.


```cpp
int main(int argc, const char ** argv) {
    struct ggml_init_params params = {
        .mem_size   = 128*1024*1024,
        .mem_buffer = NULL,
        .no_alloc   = false,
    };

    //struct ggml_opt_params opt_params = ggml_opt_default_params(GGML_OPT_ADAM);
    //opt_params.adam.alpha = 0.01f;

    struct ggml_opt_params opt_params = ggml_opt_default_params(GGML_OPT_LBFGS);

    // original threads: 8
    int nthreads = 8;
    const char *env = getenv("GGML_NTHREADS");
    if (env != NULL) {
        nthreads = atoi(env);
    }
    if (argc > 1) {
        nthreads = atoi(argv[1]);
    }
    opt_params.n_threads = nthreads;
    printf("test2: n_threads:%d\n", opt_params.n_threads);

    const float xi[] = {  1.0f,  2.0f,  3.0f,  4.0f,  5.0f , 6.0f,  7.0f,  8.0f,  9.0f,  10.0f, };
          float yi[] = { 15.0f, 25.0f, 35.0f, 45.0f, 55.0f, 65.0f, 75.0f, 85.0f, 95.0f, 105.0f, };

    const int n = sizeof(xi)/sizeof(xi[0]);

    struct ggml_context * ctx0 = ggml_init(params);

    struct ggml_tensor * x = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, n);
    struct ggml_tensor * y = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, n);

    for (int i = 0; i < n; i++) {
        ((float *) x->data)[i] = xi[i];
        ((float *) y->data)[i] = yi[i];
    }

    {
        struct ggml_tensor * t0 = ggml_new_f32(ctx0, 0.0f);
        struct ggml_tensor * t1 = ggml_new_f32(ctx0, 0.0f);

        // initialize auto-diff parameters:
        ggml_set_param(ctx0, t0);
        ggml_set_param(ctx0, t1);

        // f = sum_i[(t0 + t1*x_i - y_i)^2]/(2n)
        struct ggml_tensor * f =
            ggml_div(ctx0,
                    ggml_sum(ctx0,
                        ggml_sqr(ctx0,
                            ggml_sub(ctx0,
                                ggml_add(ctx0,
                                    ggml_mul(ctx0, x, ggml_repeat(ctx0, t1, x)),
                                    ggml_repeat(ctx0, t0, x)),
                                y)
                            )
                        ),
                    ggml_new_f32(ctx0, 2.0f*n));

        enum ggml_opt_result res = ggml_opt(NULL, opt_params, f);

        printf("t0 = %f\n", ggml_get_f32_1d(t0, 0));
        printf("t1 = %f\n", ggml_get_f32_1d(t1, 0));

        GGML_ASSERT(res == GGML_OPT_OK);

        GGML_ASSERT(is_close(ggml_get_f32_1d(t0, 0),  5.0f, 1e-3f));
        GGML_ASSERT(is_close(ggml_get_f32_1d(t1, 0), 10.0f, 1e-3f));
    }

    {
        struct ggml_tensor * t0 = ggml_new_f32(ctx0, -1.0f);
        struct ggml_tensor * t1 = ggml_new_f32(ctx0,  9.0f);

        ggml_set_param(ctx0, t0);
        ggml_set_param(ctx0, t1);

        // f = 0.5*sum_i[abs(t0 + t1*x_i - y_i)]/n
        struct ggml_tensor * f =
            ggml_mul(ctx0,
                    ggml_new_f32(ctx0, 1.0/(2*n)),
                    ggml_sum(ctx0,
                        ggml_abs(ctx0,
                            ggml_sub(ctx0,
                                ggml_add(ctx0,
                                    ggml_mul(ctx0, x, ggml_repeat(ctx0, t1, x)),
                                    ggml_repeat(ctx0, t0, x)),
                                y)
                            )
                        )
                    );


        enum ggml_opt_result res = ggml_opt(NULL, opt_params, f);

        GGML_ASSERT(res == GGML_OPT_OK);
        GGML_ASSERT(is_close(ggml_get_f32_1d(t0, 0),  5.0f, 1e-2f));
        GGML_ASSERT(is_close(ggml_get_f32_1d(t1, 0), 10.0f, 1e-2f));
    }

    {
        struct ggml_tensor * t0 = ggml_new_f32(ctx0,  5.0f);
        struct ggml_tensor * t1 = ggml_new_f32(ctx0, -4.0f);

        ggml_set_param(ctx0, t0);
        ggml_set_param(ctx0, t1);

        // f = t0^2 + t1^2
        struct ggml_tensor * f =
            ggml_add(ctx0,
                    ggml_sqr(ctx0, t0),
                    ggml_sqr(ctx0, t1)
                    );

        enum ggml_opt_result res = ggml_opt(NULL, opt_params, f);

        GGML_ASSERT(res == GGML_OPT_OK);
        GGML_ASSERT(is_close(ggml_get_f32_1d(f,  0), 0.0f, 1e-3f));
        GGML_ASSERT(is_close(ggml_get_f32_1d(t0, 0), 0.0f, 1e-3f));
        GGML_ASSERT(is_close(ggml_get_f32_1d(t1, 0), 0.0f, 1e-3f));
    }

    /////////////////////////////////////////

    {
        struct ggml_tensor * t0 = ggml_new_f32(ctx0, -7.0f);
        struct ggml_tensor * t1 = ggml_new_f32(ctx0,  8.0f);

        ggml_set_param(ctx0, t0);
        ggml_set_param(ctx0, t1);

        // f = (t0 + 2*t1 - 7)^2 + (2*t0 + t1 - 5)^2
        struct ggml_tensor * f =
            ggml_add(ctx0,
                    ggml_sqr(ctx0,
                        ggml_sub(ctx0,
                            ggml_add(ctx0,
                                t0,
                                ggml_mul(ctx0, t1, ggml_new_f32(ctx0, 2.0f))),
                            ggml_new_f32(ctx0, 7.0f)
                            )
                        ),
                    ggml_sqr(ctx0,
                        ggml_sub(ctx0,
                            ggml_add(ctx0,
                                ggml_mul(ctx0, t0, ggml_new_f32(ctx0, 2.0f)),
                                t1),
                            ggml_new_f32(ctx0, 5.0f)
                            )
                        )
                    );

        enum ggml_opt_result res = ggml_opt(NULL, opt_params, f);

        GGML_ASSERT(res == GGML_OPT_OK);
        GGML_ASSERT(is_close(ggml_get_f32_1d(f,  0), 0.0f, 1e-3f));
        GGML_ASSERT(is_close(ggml_get_f32_1d(t0, 0), 1.0f, 1e-3f));
        GGML_ASSERT(is_close(ggml_get_f32_1d(t1, 0), 3.0f, 1e-3f));
    }

    ggml_free(ctx0);

    return 0;
}

```

# `tests/test3.c`

This is a C function that appears to perform a gradient descent optimization problem on a function f(x) defined as f(x) = sum[(fj*x - l)^2]/n + lambda*|x^2|, where fj is the jth derivative of f, l is the parameter of the function, and x is the variable to be optimized. The function uses the option algorithm to minimize the risk of getting stuck in a poor local minimum. The optimization problem is solved using the services of the GNU General Library (GGML). The output of the function is a vector of floating-point numbers that represent the values of the parameters x at different points in the function domain.


```cpp
#include "ggml/ggml.h"

#include <math.h>
#include <stdio.h>
#include <stdlib.h>

bool is_close(float a, float b, float epsilon) {
    return fabs(a - b) < epsilon;
}

int main(int argc, const char ** argv) {
    struct ggml_init_params params = {
        .mem_size   = 1024*1024*1024,
        .mem_buffer = NULL,
        .no_alloc   = false,
    };

    //struct ggml_opt_params opt_params = ggml_opt_default_params(GGML_OPT_ADAM);
    struct ggml_opt_params opt_params = ggml_opt_default_params(GGML_OPT_LBFGS);

    opt_params.n_threads = (argc > 1) ? atoi(argv[1]) : 8;

    const int NP = 1 << 12;
    const int NF = 1 << 8;

    struct ggml_context * ctx0 = ggml_init(params);

    struct ggml_tensor * F = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, NF, NP);
    struct ggml_tensor * l = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, NP);

    // regularization weight
    struct ggml_tensor * lambda = ggml_new_f32(ctx0, 1e-5f);

    srand(0);

    for (int j = 0; j < NP; j++) {
        const float ll = j < NP/2 ? 1.0f : -1.0f;
        ((float *)l->data)[j] = ll;

        for (int i = 0; i < NF; i++) {
            ((float *)F->data)[j*NF + i] = ((ll > 0 && i < NF/2 ? 1.0f : ll < 0 && i >= NF/2 ? 1.0f : 0.0f) + ((float)rand()/(float)RAND_MAX - 0.5f)*0.1f)/(0.5f*NF);
        }
    }

    {
        // initial guess
        struct ggml_tensor * x = ggml_set_f32(ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, NF), 0.0f);

        ggml_set_param(ctx0, x);

        // f = sum[(fj*x - l)^2]/n + lambda*|x^2|
        struct ggml_tensor * f =
            ggml_add(ctx0,
                    ggml_div(ctx0,
                        ggml_sum(ctx0,
                            ggml_sqr(ctx0,
                                ggml_sub(ctx0,
                                    ggml_mul_mat(ctx0, F, x),
                                    l)
                                )
                            ),
                        ggml_new_f32(ctx0, (float)NP)
                        ),
                    ggml_mul(ctx0,
                        ggml_sum(ctx0, ggml_sqr(ctx0, x)),
                        lambda)
                    );

        enum ggml_opt_result res = ggml_opt(NULL, opt_params, f);

        GGML_ASSERT(res == GGML_OPT_OK);

        // print results
        for (int i = 0; i < 16; i++) {
            printf("x[%3d] = %g\n", i, ((float *)x->data)[i]);
        }
        printf("...\n");
        for (int i = NF - 16; i < NF; i++) {
            printf("x[%3d] = %g\n", i, ((float *)x->data)[i]);
        }
        printf("\n");

        for (int i = 0; i < NF; ++i) {
            if (i < NF/2) {
                GGML_ASSERT(is_close(((float *)x->data)[i],  1.0f, 1e-2f));
            } else {
                GGML_ASSERT(is_close(((float *)x->data)[i], -1.0f, 1e-2f));
            }
        }
    }

    ggml_free(ctx0);

    return 0;
}

```