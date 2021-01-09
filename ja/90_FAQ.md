このページでは、Vulkanアプリケーションの開発中に遭遇する可能性のある一般的な問題の解決策を掲載しています。

* **コアバリデーションレイヤーでアクセス違反エラーが発生します**: MSI Afterburner Service または RivaTuner Statistics Server が実行されていないことを確認してください。Vulkanと互換性に問題があります。

* **バリデーションレイヤーからのメッセージが表示されません または バリデーションレイヤーが利用できません**: まず、プログラムが終了した後にターミナルを開いたままにしておくことで、バリデーションレイヤーがエラーを表示しないか確認してください。これは Visual Studio からは F5 の代わりに Ctrl-F5 でプログラムを実行し、Linux ではターミナルウィンドウからプログラムを実行することで行うことができます。バリデーションレイヤーがオンになっており、それでもメッセージが表示されない場合は、[このページ](https://vulkan.lunarg.com/doc/view/1.2.135.0/windows/getting_started.html)の "Verify the Installation" の手順に従って、Vulkan SDKが正しくインストールされていることを確認してください。また、お使いのSDKのバージョンが1.1.106.0以上で、 `VK_LAYER_KHRONOS_validation` レイヤをサポートしていることを確認してください。

* **vkCreateSwapchainKHR が SteamOverlayVulkanLayer64.dll でエラーを引き起こします**: これは、Steam クライアントベータ版の互換性の問題のようです。いくつかの回避策があります。
    * Steam ベータプログラムからの脱退
    * 環境変数 `DISABLE_VK_LAYER_VALVE_steam_overlay_1` を `1` に設定
    * レジストリの `HKEY_LOCAL_MACHINE\SOFTWARE\Khronos\Vulkan\ImplicitLayers` にある Steam overlay Vulkan レイヤーエントリを削除

例:

![](/images/steam_layers_env.png)

