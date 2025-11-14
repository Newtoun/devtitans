
### 1\. Adicione os Imports Necessários

No topo do arquivo, junto com os outros imports, adicione:

```java
import android.content.Intent;
import android.os.UserHandle;
```

-----

### 2\. Modifique o Método `loggingLevel()`

Localize o método `private int loggingLevel()` (por volta da linha 708) e insira o novo bloco de código logo após `String[] packages = pm.getPackagesForUid(uid);`.

```java
/* Trecho de código original (linha ~708) */
        private int loggingLevel() {
            synchronized (mLock) {
                int uid = Binder.getCallingUid();

                /* ... (código original) ... */

                PackageManager pm = getContext().getPackageManager();
                String[] packages = pm.getPackagesForUid(uid);
                
                //
                // <<< INÍCIO DA SUA MODIFICAÇÃO >>>
                //
                if (packages != null) {
                    for (String pkgName : packages) {
                        if ("com.example.videoapp".equals(pkgName)) {
                            // Encontramos o app! Disparar o broadcast.
                            Slog.d(TAG, "MEU_LOG_AOSP: " + pkgName + " detectado. Disparando ACTION_ROTATION_SUGGESTION.");

                            Intent intent = new Intent("com.meu.device.ACTION_ROTATION_SUGGESTION");
                            intent.putExtra("enable", true);

                            // Precisamos enviar o broadcast como o usuário correto.
                            UserHandle user = UserHandle.getUserHandleForUid(uid);

                            // Limpamos a identidade do chamador para usar permissões de sistema
                            final long identity = Binder.clearCallingIdentity();
                            try {
                                mContext.sendBroadcastAsUser(intent, user);
                            } finally {
                                // Restauramos a identidade original
                                Binder.restoreCallingIdentity(identity);
                            }
                            // App encontrado, podemos sair do loop
                            break;
                        }
                    }
                }
                // <<< FIM DA SUA MODIFICAÇÃO >>>
                //

                /* Código original continua... */
                if (packages == null || packages.length == 0) {
                    Slog.d(TAG, "empty package from uid " + uid);
```

      * `Binder.clearCallingIdentity()`: Isso é crucial. O serviço "esquece" temporariamente que foi chamado pelo `com.example.videoapp` e assume sua própria identidade (de `system_server`), que tem permissão para enviar *broadcasts*.
      * `mContext.sendBroadcastAsUser(intent, user)`: Envia o *broadcast* para o usuário correto.
      * `Binder.restoreCallingIdentity(identity)`: Devolve a identidade original para o processo.
