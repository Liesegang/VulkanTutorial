古いグラフィックスAPIは、グラフィックスパイプラインのほとんどのステージでデフォルトの状態が設定されていました。Vulkanでは、ビューポートサイズからカラーブレンディング関数まで、すべてを明示的に指定する必要があります。この章では、これらの fixed-function オペレーションを設定するための構造体をすべて埋めていきます。

## 頂点の入力

構造体 `VkPipelineVertexInputStateCreateInfo` は、頂点シェーダに渡される頂点データのフォーマットを記述します。これは大まかに2つの部分で記述されています。

* Bindings：データ間の間隔と、データがバーテックス単位かインスタンス単位か（[instancing](https://en.wikipedia.org/wiki/Geometry_instancing)を参照）
* Attribute descriptions：頂点シェーダに渡される属性の型、どのバインディングから読み込むか、オフセットはいくらか

頂点シェーダで直接頂点データをハードコーディングしているので、とりあえず読み込む頂点データがないことを指示するように、この構造体を埋めていきます。これについては、頂点バッファの章で詳しく説明します。

```c++
VkPipelineVertexInputStateCreateInfo vertexInputInfo{};
vertexInputInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
vertexInputInfo.vertexBindingDescriptionCount = 0;
vertexInputInfo.pVertexBindingDescriptions = nullptr; // Optional
vertexInputInfo.vertexAttributeDescriptionCount = 0;
vertexInputInfo.pVertexAttributeDescriptions = nullptr; // Optional
```

`pVertexBindingDescriptions` と `pVertexAttributeDescriptions` メンバは、前述の頂点データを読み込むための詳細を記述する構造体の配列を指します。この構造体を `shaderStages` 配列の直後の `createGraphicsPipeline` 関数に追加します。

## インプットアセンブリ

`VkPipelineInputAssemblyStateCreateInfo` 構造体は、頂点からどのようなジオメトリを描画するかと、プリミティブのリスタートを有効にするかどうかという 2 つのことを記述します。前者は `topology` メンバで指定され、以下のような値を持つことができます。

* `VK_PRIMITIVE_TOPOLOGY_POINT_LIST`: 各頂点からの点を生成
* `VK_PRIMITIVE_TOPOLOGY_LINE_LIST`: 2つの頂点ごとに線を生成。頂点の再利用はしない
* `VK_PRIMITIVE_TOPOLOGY_LINE_STRIP`: 各線の終了点を次の線の開始点として利用する
* `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST`: 3つの頂点ごとに三角形を生成。頂点の再利用はしない
* `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_STRIP `: すべての三角形の 2 番目と 3 番目の頂点を次の三角形の最初の 2 つの頂点として利用

通常、頂点は頂点バッファから順にインデックスごとに読み込まれますが、*エレメントバッファ* を使えば、自分で使用するインデックスを指定することができます。これにより、頂点の再利用などの最適化を行うことができます。`primitiveRestartEnable` メンバを `VK_TRUE` に設定すると、`_STRIP` トポロジモードでは、`0xFFFF` や `0xFFFF` のような特殊なインデックスを使って、線や三角形を分割することができます。

このチュートリアルでは、三角形を描くことにしているので、構造体のデータは次のようにします。

```c++
VkPipelineInputAssemblyStateCreateInfo inputAssembly{};
inputAssembly.sType = VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO;
inputAssembly.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;
inputAssembly.primitiveRestartEnable = VK_FALSE;
```

## ビューポートとシザー

ビューポートは基本的に出力がレンダリングされるフレームバッファの領域を記述します。これはほとんどの場合、`(0, 0)`から`(width, height)`になります。そして、このチュートリアルの場合もそうです。

```c++
VkViewport viewport{};
viewport.x = 0.0f;
viewport.y = 0.0f;
viewport.width = (float) swapChainExtent.width;
viewport.height = (float) swapChainExtent.height;
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;
```

スワップチェーンとその画像のサイズは、ウィンドウの `WIDTH` や `HEIGHT` とは異なる可能性があることを覚えておいてください。スワップチェーンの画像は後にフレームバッファとして使用されるので、そのサイズを維持する必要があります。

`minDepth` と `maxDepth` は、フレームバッファに使用するデプス値の範囲を指定します。これらの値は `[0.0f, 1.0f]` の範囲内でなければなりませんが、`minDepth` は `maxDepth` より高くても構いません。特別なことをしていないのであれば、`0.0f` と `1.0f` の標準値に設定するべきです。

ビューポートが画像からフレームバッファへの変換を定義するのに対し、シザー矩形はピクセルが実際にどの領域に格納されるかを定義します。シザー矩形の外側のピクセルはラスタライザによって破棄されます。これは変換というよりはフィルターのような役割を果たします。その違いを次の図に示します。左のシザー矩形は、図のような画像が生成する可能性のある多くのもののうちの一つにすぎないことに注意してください。つまり、ビューポートよりシザー矩形が大きくさえあれば何でも良かったのです。

![](/images/viewports_scissors.png)

このチュートリアルでは、単純にフレームバッファ全体に描画したいので、フレームバッファ全体を覆うシザー矩形を指定します。

```c++
VkRect2D scissor{};
scissor.offset = {0, 0};
scissor.extent = swapChainExtent;
```

このビューポートとシザー矩形を `VkPipelineViewportStateCreateInfo` 構造体を使ってビューポートの状態に結合します。グラフィックスカードによっては複数のビューポートとシザー矩形を使用できるので、そのメンバは配列を参照します。複数のビューポートを使用するには、GPU機能を有効にする必要があります(論理デバイスの作成を参照してください)。

```c++
VkPipelineViewportStateCreateInfo viewportState{};
viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
viewportState.viewportCount = 1;
viewportState.pViewports = &viewport;
viewportState.scissorCount = 1;
viewportState.pScissors = &scissor;
```

## ラスタライザ

ラスタライザは、頂点シェーダから頂点によって形成されたジオメトリを取り込み、フラグメントシェーダによって着色されることになるフラグメントに変換します。また、[デプステスト](https://ja.wikipedia.org/wiki/Z%E3%83%90%E3%83%83%E3%83%95%E3%82%A1)、[フェイスカリング](https://en.wikipedia.org/wiki/Back-face_culling)、シザーテストを行います。ポリゴン全体を埋めるフラグメントやエッジのみを出力するように設定することもできます（ワイヤーフレームレンダリング）。これらはすべて `VkPipelineRasterizationStateCreateInfo` 構造体を使って設定します。

```c++
VkPipelineRasterizationStateCreateInfo rasterizer{};
rasterizer.sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO;
rasterizer.depthClampEnable = VK_FALSE;
```

`depthClampEnable` が `VK_TRUE` に設定されている場合、近傍平面と遠方平面を越えたフラグメントは破棄されるのではなく、クランプされます。これはシャドウマップのような特殊な場合に便利です。これを使うにはGPU機能を有効にする必要がある。

```c++
rasterizer.rasterizerDiscardEnable = VK_FALSE;
```

`rasterizerDiscardEnable` が `VK_TRUE` に設定されている場合、ジオメトリはラスタライザステージを通過しません。これは基本的にフレームバッファへの一切の出力を無効にします。

```c++
rasterizer.polygonMode = VK_POLYGON_MODE_FILL;
```

ポリゴンモード`は、ジオメトリからフラグメントをどのように生成するかを決定します。以下のモードが利用可能です。

* `VK_POLYGON_MODE_FILL`: ポリゴンの領域をフラグメントで埋めます。
* `VK_POLYGON_MODE_LINE`: ポリゴンのエッジを線で描画します。
* `VK_POLYGON_MODE_POINT`: ポリゴンの頂点は点として描画されます。

fill以外のモードを使用するには、GPU機能を有効にする必要があります。

```c++
rasterizer.lineWidth = 1.0f;
```

`lineWidth` メンバは簡単であり、線の太さをフラグメントの数で表します。サポートされる最大線幅はハードウェアに依存し、`1.0f` より太い線は `wideLines` GPU 機能を有効にする必要があります。

```c++
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_CLOCKWISE;
```

`cullMode` メンバは、使用するフェイスカリングのタイプを決定します。カリングを無効にするか、表向きの面をカリングするか、裏向きの面をカリングするか、あるいはその両方を指定することができます。`frontFace` 変数は、正面を向いているとみなされる面の頂点の順序を指定します。時計回りまたは反時計回りが指定できます。

```c++
rasterizer.depthBiasEnable = VK_FALSE;
rasterizer.depthBiasConstantFactor = 0.0f; // Optional
rasterizer.depthBiasClamp = 0.0f; // Optional
rasterizer.depthBiasSlopeFactor = 0.0f; // Optional
```

ラスタライザは、一定の値を追加したり、フラグメントの傾きに基づいてバイアスをかけたりすることで、デプス値を変更することができます。これはシャドウマッピングに使われることもありますが、ここでは使いません。単に `depthBiasEnable` を `VK_FALSE` に設定します。

## マルチサンプリング

構造体 `VkPipelineMultisampleStateCreateInfo` はアンチエイリアス処理を行う方法の一つであるマルチサンプリングを設定します。これは、同じピクセルにラスタライズする複数のポリゴンのフラグメントシェーダの結果を組み合わせることで動作します。これは主にエッジに沿って発生しますが、これは最も顕著なエイリアシングアーチファクトが発生する場所でもあります。1 つのピクセルに 1 つのポリゴンだけがマップされている場合、フラグメントシェーダを何度も実行する必要がないため、単に高解像度にレンダリングしてからダウンスケーリングするよりも大幅にコストが削減されます。この機能を有効にするには、GPU 機能を有効にする必要があります。
The `VkPipelineMultisampleStateCreateInfo` struct configures multisampling, which is one of the ways to perform [anti-aliasing](https://en.wikipedia.org/wiki/Multisample_anti-aliasing). It works by combining the fragment shader results of multiple polygons that rasterize to the same pixel. This mainly occurs along edges, which is also where the most noticeable aliasing artifacts occur. Because it doesn't need to run the fragment shader multiple times if only one polygon maps to a pixel, it is significantly less expensive than simply rendering to a higher resolution and then downscaling. Enabling it requires enabling a GPU feature.

```c++
VkPipelineMultisampleStateCreateInfo multisampling{};
multisampling.sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO;
multisampling.sampleShadingEnable = VK_FALSE;
multisampling.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;
multisampling.minSampleShading = 1.0f; // Optional
multisampling.pSampleMask = nullptr; // Optional
multisampling.alphaToCoverageEnable = VK_FALSE; // Optional
multisampling.alphaToOneEnable = VK_FALSE; // Optional
```

マルチサンプリングについては後の章で説明しますが、今のところは無効にしておきましょう。

## デプステストとステンシルテスト

デプスやステンシルバッファを使用している場合は、`VkPipelineDepthStencilStateCreateInfo` を使用してデプスやステンシルのテストを設定する必要があります。今はないので、このような構造体へのポインタの代わりに `nullptr` を渡すだけです。これについてはデプスバッファの章で説明します。

## カラーブレンド

フラグメントシェーダが返した色を、すでにフレームバッファにある色と組み合わせる必要があります。この変換はカラーブレンディングとして知られており、2つの方法があります。

* 新旧の値を混ぜて最終的な色を作る
* ビット演算を使用して、新旧の値を結合する

カラーブレンドを設定するための構造体には2種類あります。1 つ目の構造体 `VkPipelineColorBlendAttachmentState` には、アタッチされたフレームバッファごとの設定であり、2 つ目の構造体 `VkPipelineColorBlendStateCreateInfo` には *グローバル* カラーブレンディングの設定です。今回のケースでは、フレームバッファは1つしかありません。

```c++
VkPipelineColorBlendAttachmentState colorBlendAttachment{};
colorBlendAttachment.colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT;
colorBlendAttachment.blendEnable = VK_FALSE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD; // Optional
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD; // Optional
```

このレームバッファごとの構造体を使用すると、カラーブレンドの１つ目の方法を設定することができます。以下の疑似コードが実行される操作を最もよく表しています。

```c++
if (blendEnable) {
    finalColor.rgb = (srcColorBlendFactor * newColor.rgb) <colorBlendOp> (dstColorBlendFactor * oldColor.rgb);
    finalColor.a = (srcAlphaBlendFactor * newColor.a) <alphaBlendOp> (dstAlphaBlendFactor * oldColor.a);
} else {
    finalColor = newColor;
}

finalColor = finalColor & colorWriteMask;
```

`blendEnable` が `VK_FALSE` に設定されている場合、フラグメントシェーダの新しい色がそのまま渡されます。そうでない場合は、新しい色を計算するために2つの色をブレンドする操作が行われます。結果として得られた色は `colorWriteMask` と AND が取られ、実際にどのチャンネルが通過するかが決定されます。

カラーブレンディングを使う最も一般的な方法はアルファブレンディングを実装することです。アルファブレンディングでは、透明度に応じて新しい色が古い色とブレンドされます。そして `finalColor` は次のように計算されます。

```c++
finalColor.rgb = newAlpha * newColor + (1 - newAlpha) * oldColor;
finalColor.a = newAlpha.a;
```

これは、以下のパラメータで実現できます。

```c++
colorBlendAttachment.blendEnable = VK_TRUE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_SRC_ALPHA;
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA;
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD;
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD;
```

仕様書の `VkBlendFactor` と `VkBlendOp` の列挙体には、可能なすべての操作が記載されています。

2 番目の構造体は、すべてのフレームバッファの構造体の配列を参照し、前述の計算でブレンド係数として使用できるブレンド定数を設定することができます。

```c++
VkPipelineColorBlendStateCreateInfo colorBlending{};
colorBlending.sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO;
colorBlending.logicOpEnable = VK_FALSE;
colorBlending.logicOp = VK_LOGIC_OP_COPY; // Optional
colorBlending.attachmentCount = 1;
colorBlending.pAttachments = &colorBlendAttachment;
colorBlending.blendConstants[0] = 0.0f; // Optional
colorBlending.blendConstants[1] = 0.0f; // Optional
colorBlending.blendConstants[2] = 0.0f; // Optional
colorBlending.blendConstants[3] = 0.0f; // Optional
```

2番目のブレンド方法(ビット演算によるブレンド)を使いたい場合は、`logicOpEnable` を `VK_TRUE` に設定します。`logicOp` フィールドでビット演算を指定できます。これにより、アタッチされている各フレームバッファの `blendEnable` が `VK_FALSE` に設定されているかのようになり、1番目の方法が自動的に無効になることに注意してください。また、このモードでは `colorWriteMask` が使用され、フレームバッファ内のどのチャンネルが実際に影響を受けるかを決定します。今回のように、両方のモードを無効にすることも可能で、その場合フラグメントの色はそのままフレームバッファに書き込まれます。

## ダイナミックステート

前の構造体で指定した状態の一部は、パイプラインを再作成せずに変更することができます。例としては、ビューポートのサイズ、線幅、ブレンド定数などが挙げられます。そのためには、以下のように `VkPipelineDynamicStateCreateInfo` 構造体を埋めます。

```c++
VkDynamicState dynamicStates[] = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_LINE_WIDTH
};

VkPipelineDynamicStateCreateInfo dynamicState{};
dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
dynamicState.dynamicStateCount = 2;
dynamicState.pDynamicStates = dynamicStates;
```

これにより、これらの値の設定が無視され、描画時にデータの指定が必要になります。これについては後の章で説明します。この構造体は、動的な状態を持たない場合は、 `nullptr` を指定します。

## パイプラインレイアウト

シェーダでは `uniform` 値を使用することができます。これはダイナミックステート変数に似たグローバルで、描画時に変更することで、シェーダを再作成しなくてもシェーダの動作を変更することができます。これは、頂点シェーダに変換行列を渡したり、フラグメントシェーダでテクスチャサンプラーを作成したりするのによく使われます。

これらのユニフォーム値は、パイプラインの作成時に `VkPipelineLayout` オブジェクトを作成して指定する必要があります。後の章まで使用することはありませんが、その場合でも空のパイプラインレイアウトを作成する必要があります。

後で他の関数から参照することになるので、このオブジェクトを保持するクラスメンバを作成します。

```c++
VkPipelineLayout pipelineLayout;
```

そして、`createGraphicsPipeline`関数内でオブジェクトを作成します。

```c++
VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 0; // Optional
pipelineLayoutInfo.pSetLayouts = nullptr; // Optional
pipelineLayoutInfo.pushConstantRangeCount = 0; // Optional
pipelineLayoutInfo.pPushConstantRanges = nullptr; // Optional

if (vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &pipelineLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create pipeline layout!");
}
```

この構造体はまた、*プッシュ定数*も指定しています。これは、動的な値をシェーダに渡すもう一つの方法です。今後の章で説明します。パイプラインレイアウトはプログラムの寿命を通して参照されるので、最後に破棄してください。

```c++
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```

## 最後に

fixed-function の設定はこれですべて終了です！このすべてをゼロから設定するのは大変な作業ですが、グラフィックパイプラインで起こっていることをほぼ完全に理解できるようになりました！これにより、特定のコンポーネントのデフォルト状態が期待していたものと異なるため、予期せぬ動作をする可能性が減ります。

しかし、最終的にグラフィックパイプラインを作成する前に、もう一つ作成しなければならないオブジェクトがあります。それは、[レンダーパス](!ja/三角形を描く/グラフィックスパイプラインの基礎/レンダーパス)です

[C++ コード](/code/10_fixed_functions.cpp) /
[頂点シェーダ](/code/09_shader_base.vert) /
[フラグメントシェーダ](/code/09_shader_base.frag)
