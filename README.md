# AOSP: Anulação da Trava de Rotação via Gatilho Externo

Este documento detalha o processo de modificação do AOSP para criar um mecanismo que **habilita temporariamente a rotação automática (baseada em sensor)**, mesmo que o usuário a tenha desativado nas configurações do sistema.

## 1\. Objetivo

Diferente da implementação anterior que forçava e travava a tela em um ângulo fixo, o objetivo desta modificação é mais sutil: usar um gatilho externo (uma `Intent` via `adb`) para "ligar" ou "desligar" a rotação automática do dispositivo, ignorando a preferência do usuário.

Isso é útil em cenários onde um evento específico (ex: conectar um acessório, iniciar um modo especial) deve permitir que o dispositivo gire livremente, retornando ao comportamento padrão quando o evento termina.

## 2\. Estratégia de Implementação

A solução se baseia em criar um "interruptor de anulação" (`override switch`) dentro da lógica central de rotação do Android.

1.  **Criar o Interruptor:** Adicionamos uma variável booleana estática (`sRotationOverride`) na classe `com.android.internal.view.RotationPolicy`. Quando `true`, a rotação automática é forçada.
2.  **Modificar o Ponto de Verificação:** Alteramos o método `RotationPolicy.isRotationLocked()` para que ele verifique o estado do nosso interruptor **antes** de consultar a configuração do usuário. Se o interruptor estiver ligado, o método informa ao sistema que a rotação não está travada.
3.  **Controlar o Interruptor:** Criamos métodos públicos para ligar e desligar o interruptor.
4.  **Adaptar o Gatilho:** O `BroadcastReceiver` que já havíamos criado foi modificado para, em vez de enviar um ângulo, enviar um comando para ligar ou desligar o interruptor de anulação.

## 3\. Modificações no Código-Fonte

### Passo 1: O Coração da Lógica (`RotationPolicy.java`)

Esta é a alteração mais crítica. Introduzimos o nosso interruptor e a lógica de anulação diretamente na classe que o sistema consulta para saber se a rotação está travada.

  * **Arquivo Modificado:** `frameworks/base/core/java/com/android/internal/view/RotationPolicy.java`

  * **Modificações:**

    ```java
    // ...
    import android.provider.Settings;
    import android.util.Slog; // Import para logs

    public final class RotationPolicy {
        // ...
        private static final String TAG = "RotationPolicy";

        // 1. Nosso interruptor de anulação. 'volatile' para visibilidade entre threads.
        private static volatile boolean sRotationOverride = false;

        // ...

        public static boolean isRotationLocked(Context context) {
            // 2. Lógica de anulação: se o interruptor estiver ligado, ignoramos o usuário.
            if (sRotationOverride) {
                Slog.d(TAG, "isRotationLocked: Rotação anulada, retornando 'false' (desbloqueado).");
                return false;
            }

            // Lógica original do AOSP
            return Settings.System.getIntForUser(context.getContentResolver(),
                    Settings.System.ACCELEROMETER_ROTATION, 0, UserHandle.USER_CURRENT) == 0;
        }

        // ...

        /**
         * 3. Método público para controlar nosso interruptor a partir de outras partes do sistema.
         * Anula a configuração de rotação do usuário, forçando a habilitação da rotação automática.
         * @param enabled true para habilitar a rotação automática, false para retornar ao padrão.
         */
        public static void setRotationOverride(boolean enabled) {
            Slog.d(TAG, "setRotationOverride: " + enabled);
            sRotationOverride = enabled;
        }

        // ...
    }
    ```

### Passo 2: Atualização da Classe Controladora (`CustomRotationController.java`)

Nossa classe de ajuda foi simplificada para apenas repassar os comandos de habilitar/desabilitar para a `RotationPolicy`.

  * **Arquivo Modificado:** `frameworks/base/services/core/java/com/android/server/CustomRotationController.java`

  * **Código Atualizado:**

    ```java
    package com.android.server;

    import android.content.Context;
    import android.util.Slog;
    import com.android.internal.view.RotationPolicy;

    /**
     * Classe para controlar a anulação da política de rotação automática.
     */
    public class CustomRotationController {
        private static final String TAG = "CustomRotationController";

        /**
         * Habilita a anulação, forçando a rotação automática a ser ativada.
         */
        public static void enableAutoRotationOverride(Context context) {
            Slog.d(TAG, "Habilitando a anulação da rotação automática.");
            RotationPolicy.setRotationOverride(true);
        }

        /**
         * Desabilita a anulação, retornando o controle à configuração do usuário.
         */
        public static void disableAutoRotationOverride(Context context) {
            Slog.d(TAG, "Desabilitando a anulação da rotação automática.");
            RotationPolicy.setRotationOverride(false);
        }
    }
    ```

### Passo 3: Adaptação do Gatilho (`SystemServer.java`)

O `BroadcastReceiver` agora interpreta um extra booleano para decidir se deve ligar ou desligar o nosso interruptor.

  * **Arquivo Modificado:** `frameworks/base/services/java/com/android/server/SystemServer.java`

  * **Modificação (dentro do método `onReceive`):**

    ```java
    @Override
    public void onReceive(Context context, Intent intent) {
        if ("com.meu.device.ACTION_FORCE_ROTATE".equals(intent.getAction())) {
            // Pega um extra booleano 'enable'. O padrão é 'false' se não for fornecido.
            boolean shouldEnable = intent.getBooleanExtra("enable", false);
            
            if (shouldEnable) {
                com.android.server.CustomRotationController.enableAutoRotationOverride(context);
            } else {
                com.android.server.CustomRotationController.disableAutoRotationOverride(context);
            }
        }
    }
    ```

## 4\. Uso e Teste

Após compilar e instalar a nova imagem, use os seguintes comandos `adb` para controlar a anulação da rotação.

  * **Para HABILITAR a Rotação Automática (Ignora o Usuário):**
    O dispositivo começará a girar com base nos sensores, mesmo que a opção "Rotação automática" esteja desativada na UI.

    ```bash
    adb shell am broadcast -a com.meu.device.ACTION_FORCE_ROTATE --ez enable true
    ```

  * **Para DESABILITAR a Anulação (Retorna ao Comportamento Padrão):**
    O dispositivo voltará a respeitar a configuração do usuário. Se a rotação estava desativada, ele travará na orientação atual.

    ```bash
    adb shell am broadcast -a com.meu.device.ACTION_FORCE_ROTATE --ez enable false
    ```

    > O flag `--ez` é usado para enviar um extra do tipo booleano (`boolean`) via `adb`.

## 5\. Depuração e Verificação

Use o `logcat` para monitorar a funcionalidade.

1.  **Verificar o Estado da Anulação:** Filtre pela `TAG` da `RotationPolicy` para ver o nosso interruptor em ação.

    ```bash
    adb logcat -s "RotationPolicy"
    ```

    Ao enviar os comandos `adb`, você verá as mensagens "setRotationOverride: true/false" e, ao girar o dispositivo, a mensagem "isRotationLocked: Rotação anulada...".

2.  **Verificar o Gatilho:** Para garantir que o `BroadcastReceiver` está recebendo o comando e chamando o controlador, filtre pela `TAG` `CustomRotationController`.

    ```bash
    adb logcat -s "CustomRotationController"
    ```

   ```java
   A cada comando `adb`, uma mensagem de "Habilitando..." ou "Desabilitando..." deve aparecer.

    public void onRotationProposal(int rotation, boolean isValid) {
        // ... (as primeiras verificações 'isUserSetupComplete', 'OEM_DISALLOW_ROTATION_IN_SUW'
        //      e 'mRotationButton.acceptRotationProposal()' continuam iguais)

        int windowRotation = mWindowRotationProvider.get();

        // ... (a verificação 'mHomeRotationEnabled' continua igual)

        // Se a proposta não for válida, esconde o botão (comportamento original)
        if (!isValid) {
            setRotateSuggestionButtonState(false /* visible */);
            return;
        }

        // Se a rotação já for a sugerida, esconde o botão (comportamento original)
        if (rotation == windowRotation) {
            mMainThreadHandler.removeCallbacks(mRemoveRotationProposal);
            setRotateSuggestionButtonState(false /* visible */);
            return;
        }

        // Armazena a sugestão (comportamento original)
        Log.i(TAG, "onRotationProposal(rotation=" + rotation + ")");
        mLastRotationSuggestion = rotation; // Lembra a rotação
        
        // ... (a lógica original para 'mIconResId' e 'mRotationButton.updateIcon'
        //      pode continuar aqui, linhas 514-519)
        final boolean rotationCCW = Utilities.isRotationAnimationCCW(windowRotation, rotation);
        if (windowRotation == Surface.ROTATION_0 || windowRotation == Surface.ROTATION_180) {
            mIconResId = rotationCCW ? mIconCcwStart0ResId : mIconCwStart0ResId;
        } else { // 90 or 270
            mIconResId = rotationCCW ? mIconCcwStart90ResId : mIconCwStart90ResId;
        }
        mRotationButton.updateIcon(mLightIconColor, mDarkIconColor);


        if (true) {
            Log.i(TAG, "mMinhaFlagDeAutoRotacao=true. Rotacionando automaticamente.");
            
            setRotationLockedAtAngle(1) ;            
            mUiEventLogger.log(RotationButtonEvent.ROTATION_SUGGESTION_ACCEPTED);
            
            return; 
        }


        if (canShowRotationButton()) {
            // The navbar is visible / it's in visual immersive mode, so show the icon right away
            showAndLogRotationSuggestion();
        } else {
            // If the navbar isn't shown, flag the rotate icon to be shown should the navbar become
            // visible given some time limit.
            mPendingRotationSuggestion = true;
            mMainThreadHandler.removeCallbacks(mCancelPendingRotationProposal);
            mMainThreadHandler.postDelayed(mCancelPendingRotationProposal,
                    NAVBAR_HIDDEN_PENDING_ICON_TIMEOUT_MS);
        }
    }
```
