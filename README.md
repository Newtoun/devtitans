
### 1\. Alterações em `SystemServer.java`

Precisamos de adicionar um `Handler` e um `Runnable` dentro do `BroadcastReceiver` para gerir a contagem dos 30 segundos.

**Novos Imports Necessários:**

```java
import android.os.Handler;
import android.os.Looper;
```

**Código Atualizado para inserir em `startOtherServices()`:**

```java
// ... dentro de startOtherServices(), após a inicialização de mPowerManagerService ...

/* * INÍCIO DA IMPLEMENTAÇÃO CUSTOMIZADA: Auto-aceite de rotação 
 * Registra um BroadcastReceiver com TIMER DE 30 SEGUNDOS.
 */
try {
    Slog.i(TAG, "Registrando Custom Rotation Receiver com Timer");

    IntentFilter filter = new IntentFilter("com.meu.device.ACTION_ROTATION_SUGGESTION");
    Context systemContext = mSystemContext;
    
    // Cria um Handler associado à Looper principal para gerir o tempo
    final Handler mHandler = new Handler(Looper.getMainLooper());

    systemContext.registerReceiverAsUser(new BroadcastReceiver() {
        
        // Armazena a referência para a tarefa de desligar para podermos cancelar se necessário
        private Runnable mDisableTask;

        @Override
        public void onReceive(Context context, Intent intent) {
            if ("com.meu.device.ACTION_ROTATION_SUGGESTION".equals(intent.getAction())) {

                boolean shouldEnable = intent.getBooleanExtra("enable", false);
                
                // Obtém o ID do utilizador atual
                int currentUserId = mActivityManagerService.getCurrentUserId();

                if (shouldEnable) {
                    // 1. ATIVAR: Define como 1
                    Slog.i(TAG, "Rotation Auto-Accept: ATIVADO para o User " + currentUserId);
                    Settings.System.putIntForUser(context.getContentResolver(),
                            "auto_accept_rotation_suggestion",
                            1,
                            currentUserId);

                    // 2. LIMPAR: Se já existia um timer a correr, cancela-o (reset ao tempo)
                    if (mDisableTask != null) {
                        mHandler.removeCallbacks(mDisableTask);
                    }

                    // 3. AGENDAR: Cria a tarefa para desligar daqui a 30 segundos
                    mDisableTask = new Runnable() {
                        @Override
                        public void run() {
                            Slog.i(TAG, "Rotation Auto-Accept: DESATIVADO por Timeout (30s) para o User " + currentUserId);
                            Settings.System.putIntForUser(context.getContentResolver(),
                                    "auto_accept_rotation_suggestion",
                                    0,
                                    currentUserId);
                        }
                    };

                    // 30000 milissegundos = 30 segundos
                    mHandler.postDelayed(mDisableTask, 30000);

                } else {
                    // Se o comando for explícito para DESATIVAR (enable=false)
                    Slog.i(TAG, "Rotation Auto-Accept: DESATIVADO manual para o User " + currentUserId);
                    Settings.System.putIntForUser(context.getContentResolver(),
                            "auto_accept_rotation_suggestion",
                            0,
                            currentUserId);

                    // Cancela o timer pendente, pois já desligámos manualmente
                    if (mDisableTask != null) {
                        mHandler.removeCallbacks(mDisableTask);
                        mDisableTask = null;
                    }
                }
            }
        }
    }, UserHandle.ALL, filter, null, null, Context.RECEIVER_EXPORTED);

} catch (Exception e) {
    Slog.e(TAG, "Falha ao registrar Custom Rotation Receiver", e);
}
/* FIM DA IMPLEMENTAÇÃO CUSTOMIZADA */
```

### O que mudou na lógica:

1.  **Handler:** Criámos um `new Handler(Looper.getMainLooper())`. O Handler permite executar código no futuro (`postDelayed`).
2.  **Lógica ao Receber `enable = true`:**
      * Ativa a flag imediatamente (põe a 1).
      * **Remove** qualquer timer anterior (caso o utilizador envie o comando várias vezes seguidas, o tempo reinicia do zero em vez de desligar a meio).
      * **Agenda** um `Runnable` para correr daqui a 30.000ms (30 segundos). Esse `Runnable` voltará a colocar a flag a 0.
3.  **Lógica ao Receber `enable = false`:**
      * Desativa a flag imediatamente.
      * Cancela o timer pendente (para evitar que o timer tente desligar algo que já está desligado).

### 2\. Alterações em `RotationButtonController.java`

**Não precisa de fazer nada aqui.**

Como o `RotationButtonController` já tem o `SettingsObserver` configurado (graças à sua excelente implementação com as correções de bug), ele detetará automaticamente quando o `SystemServer` alterar o valor de volta para `0` após 30 segundos e deixará de aceitar a rotação.

### Resumo

Esta abordagem é a mais robusta porque:

1.  Centraliza a lógica de controlo no sistema (`SystemServer`).
2.  Garante que o timer sobrevive mesmo que a SystemUI reinicie.
3.  Previne condições de corrida (race conditions) reiniciando o contador se o comando for enviado múltiplas vezes.
