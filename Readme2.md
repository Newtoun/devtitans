
### 1\. Modificação no `SystemServer.java`

Aqui, o `BroadcastReceiver` (que você já tinha de um exemplo anterior) não vai mais chamar uma classe controladora. Ele vai escrever diretamente nas configurações do sistema.

**Arquivo:** `frameworks/base/services/java/com/android/server/SystemServer.java`

```java
@Override
public void onReceive(Context context, Intent intent) {
    // ... outras intents
    
    // 1. Usar a action "rotationSuggestion" que você pediu
    if ("com.meu.device.ACTION_ROTATION_SUGGESTION".equals(intent.getAction())) {
        
        // 2. Pegar o valor booleano do 'adb'
        boolean shouldEnable = intent.getBooleanExtra("enable", false);
        int settingValue = shouldEnable ? 1 : 0;
        
        Log.d(TAG, "Recebido ACTION_ROTATION_SUGGESTION, definindo valor para: " + settingValue);

        // 3. A MÁGICA: Escrever no banco de dados de Settings
        //    Usamos "putIntForUser" para que funcione corretamente em múltiplos usuários.
        Settings.System.putIntForUser(context.getContentResolver(),
                "auto_accept_rotation_suggestion", // O nome da nossa nova "flag"
                settingValue,
                UserHandle.USER_CURRENT);
    }
}
```

-----

### 2\. Modificação no `RotationButtonController.java`

Este arquivo agora precisa de três coisas:

1.  Uma variável local para guardar o estado (`mAutoAcceptSuggestion`).
2.  Um `ContentObserver` para atualizar essa variável quando o `SystemServer` a alterar.
3.  A lógica em `onRotationProposal` que lê a variável local.

**Arquivo:** `frameworks/base/packages/SystemUI/shared/src/com/android/systemui/shared/rotation/RotationButtonController.java`

```java
// ...
import android.content.ContentResolver;
import android.database.ContentObserver;
import android.net.Uri;
import android.os.Handler;
import android.os.UserHandle;
import android.provider.Settings;
// ...

public class RotationButtonController {
    // ...
    private static final String TAG = "RotationButtonController";
    
    // 1. A constante para o nome da nossa flag (deve ser igual à do SystemServer)
    private static final String AUTO_ACCEPT_SETTING_NAME = "auto_accept_rotation_suggestion";

    // ...
    private final Context mContext;
    private final Handler mMainThreadHandler = new Handler(Looper.getMainLooper());
    // ...
    
    // 2. A variável de estado local
    private boolean mAutoAcceptSuggestion = false;

    // 3. O nosso observador
    private SettingsObserver mSettingsObserver;

    // ... (Construtor)
    public RotationButtonController(Context context, ...) {
        mContext = context;
        // ...
        
        // 4. Instanciar o observador
        mSettingsObserver = new SettingsObserver(mMainThreadHandler);
    }

    public void init() {
        registerListeners(true /* registerRotationWatcher */);
        // ...

        // 5. Registrar o observador para começar a "assistir"
        mSettingsObserver.register();
        // Atualiza o valor na inicialização
        mSettingsObserver.onChange(false, null);
    }

    public void onDestroy() {
        unregisterListeners();
        
        // 6. Parar de "assistir"
        mSettingsObserver.unregister();
    }

    // ...

    public void onRotationProposal(int rotation, boolean isValid) {
        // ... (Toda a lógica original de verificação)

        // ... (Lógica de 'mLastRotationSuggestion' e 'mRotationButton.updateIcon')
        mRotationButton.updateIcon(mLightIconColor, mDarkIconColor);

        // 7. A LÓGICA FINAL (lendo a variável local, que é rápida)
        if (mAutoAcceptSuggestion) {
            Log.i(TAG, "mAutoAcceptSuggestion=true. Rotacionando automaticamente.");
            
            setRotationLockedAtAngle(
                    RotationPolicyUtil.isRotationLocked(mContext),
                    rotation, 
                    "AutoAcceptSuggestion");
            
            mUiEventLogger.log(RotationButtonEvent.ROTATION_SUGGESTION_ACCEPTED);
            incrementNumAcceptedRotationSuggestionsIfNeeded(); 
            
            return; // Pula a exibição do botão
        }

        if (canShowRotationButton()) {
            // ... (resto do método original)
        }
    }

    // 8. Definição da classe interna do ContentObserver
    private class SettingsObserver extends ContentObserver {
        private final Uri mUri = Settings.System.getUriFor(AUTO_ACCEPT_SETTING_NAME);

        SettingsObserver(Handler handler) {
            super(handler);
        }

        public void register() {
            mContext.getContentResolver().registerContentObserver(
                    mUri, false, this, UserHandle.USER_ALL);
        }

        public void unregister() {
            mContext.getContentResolver().unregisterContentObserver(this);
        }

        @Override
        public void onChange(boolean selfChange, Uri uri) {
            // 9. Ocorreu uma mudança! Ler o novo valor e atualizar a variável local.
            ContentResolver r = mContext.getContentResolver();
            int value = Settings.System.getIntForUser(r,
                    AUTO_ACCEPT_SETTING_NAME, 0, UserHandle.USER_CURRENT);
            
            mAutoAcceptSuggestion = (value == 1);
            
            Log.d(TAG, "SettingsObserver: mAutoAcceptSuggestion atualizado para " + mAutoAcceptSuggestion);
        }
    }
    
    // ... (Resto da classe)
}
```

### Vantagens desta Abordagem

1.  **Desacoplamento:** `SystemUI` não precisa saber *quem* ou *por que* a flag mudou. Ele apenas reage à mudança.
2.  **Eficiência:** `onRotationProposal` (que é chamado frequentemente) apenas lê uma variável booleana local (`mAutoAcceptSuggestion`), o que é instantâneo. A leitura do banco de dados (que é mais lenta) só acontece quando o valor realmente muda, graças ao `ContentObserver`.
3.  **Robustez:** Este é o padrão do AOSP para comunicação de estado entre o `SystemServer` e outros componentes do sistema, como o `SystemUI`.

Claro. Esta é uma arquitetura comum para permitir que o `SystemServer` notifique outros aplicativos (ou o próprio `SystemUI`) sobre eventos específicos de reprodução de mídia.

Aqui está a documentação em markdown que detalha como modificar o `MediaSessionService` para disparar um broadcast `Intent` quando o estado de reprodução de um pacote específico (genérico) é alterado.

-----

## AOSP: Disparar Intent na Mudança de Estado de Reprodução de Pacote

Este documento descreve como modificar o `MediaSessionService` para monitorar mudanças no estado de reprodução (como play, pause, etc.) de um pacote de aplicativo específico e, em resposta, disparar um broadcast `Intent` para todo o sistema.

### 1\. Objetivo

O objetivo é adicionar um "gatilho" (trigger) dentro do `MediaSessionService` que:

1.  Identifica quando uma sessão de mídia pertence a um pacote-alvo (ex: `"com.example.generic"`).
2.  Detecta qualquer mudança em seu estado de reprodução (via `onSessionPlaybackStateChanged`).
3.  Envia uma `Intent` de broadcast com uma ação customizada (ex: `com.meu.device.ACTION_GENERIC_PLAYBACK_STATE_CHANGED`), contendo detalhes sobre a mudança de estado.

### 2\. Estratégia de Implementação

1.  **Definir Constantes:** Adicionar constantes de classe em `MediaSessionService.java` para o nome do pacote-alvo e a ação da `Intent` a ser disparada.
2.  **Adicionar Imports:** Garantir que `android.content.Intent`, `android.os.UserHandle`, e `android.os.Binder` estejam importados.
3.  **Modificar `onSessionPlaybackStateChanged`:** Inserir a lógica de verificação de pacote e o disparo da `Intent` dentro deste método. A `Intent` será enviada usando `mContext.sendBroadcastAsUser` para garantir que ela seja entregue no perfil de usuário correto.

### 3\. Modificações no Código-Fonte

**Arquivo:** `frameworks/base/services/media/java/com/android/server/media/MediaSessionService.java`

#### Passo 1: Adicionar Imports

Certifique-se de que os seguintes `imports` estão presentes no início do arquivo:

```java
// ...
import android.content.Context;
import android.content.Intent; // <<< ADICIONAR
import android.content.IntentFilter;
// ...
import android.os.Binder; // <<< ADICIONAR
import android.os.Bundle;
import android.os.Handler;
// ...
import android.os.UserHandle; // <<< ADICIONAR
import android.os.UserManager;
// ...
```

#### Passo 2: Definir Constantes de Classe

Adicione estas constantes perto das outras (logo após a definição de `TAG` e `DEBUG`):

```java
public class MediaSessionService extends SystemService implements Monitor {
    private static final String TAG = "MediaSessionService";
    static final boolean DEBUG = Log.isLoggable(TAG, Log.DEBUG);
    // ...

    // --- INÍCIO DA MODIFICAÇÃO ---
    // 1. Defina o nome do pacote-alvo que você deseja monitorar
    private static final String TARGET_PACKAGE_NAME = "com.example.generic"; // Mude para o seu pacote

    // 2. Defina a ação da Intent que será disparada
    private static final String ACTION_TARGET_PLAYBACK_STATE_CHANGED = 
            "com.meu.device.ACTION_GENERIC_PLAYBACK_STATE_CHANGED";
    // --- FIM DA MODIFICAÇÃO ---

    private static final int WAKELOCK_TIMEOUT = 5000;
    // ...
```

#### Passo 3: Modificar o Método `onSessionPlaybackStateChanged`

Localize o método `onSessionPlaybackStateChanged` e adicione o novo bloco de código no final do bloco `synchronized (mLock)`.

```java
    void onSessionPlaybackStateChanged(
            MediaSessionRecordImpl record,
            boolean shouldUpdatePriority,
            @Nullable PlaybackState playbackState) {
        synchronized (mLock) {
            FullUserRecord user = getFullUserRecordLocked(record.getUserId());
            if (user == null || !user.mPriorityStack.contains(record)) {
                Log.d(TAG, "Unknown session changed playback state. Ignoring.");
                return;
            }
            user.mPriorityStack.onPlaybackStateChanged(record, shouldUpdatePriority);
            boolean isUserEngaged = isUserEngaged(record, playbackState);
            Log.d(
                    TAG,
                    "onSessionPlaybackStateChanged:"
                            + " record="
                            + record
                            + " playbackState="
                            + playbackState);
            reportMediaInteractionEvent(record, isUserEngaged);

            // --- INÍCIO DA MODIFICAÇÃO ---

            // 3. Verificar se o pacote da sessão é o nosso pacote-alvo
            if (TARGET_PACKAGE_NAME.equals(record.getPackageName())) {
                Log.i(TAG, "Pacote-alvo (" + TARGET_PACKAGE_NAME + ") mudou o estado. Disparando Intent.");

                // 4. Criar a Intent
                Intent intent = new Intent(ACTION_TARGET_PLAYBACK_STATE_CHANGED);
                intent.putExtra("packageName", record.getPackageName());
                intent.putExtra("isUserEngaged", isUserEngaged); // Envia o estado (true = tocando, false = pausado/parado)
                if (playbackState != null) {
                    intent.putExtra("playbackState", playbackState.getState()); // Envia o estado numérico (ex: 3 = PLAYING)
                }
                // Adicionar flags para garantir que receptores em background recebam
                intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND | Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND);

                // 5. Enviar o broadcast dentro do perfil de usuário da sessão
                //    (Necessário limpar a identidade da chamada, pois estamos no SystemServer)
                final long token = Binder.clearCallingIdentity();
                try {
                    mContext.sendBroadcastAsUser(intent, UserHandle.of(record.getUserId()));
                } finally {
                    Binder.restoreCallingIdentity(token);
                }
            }
            // --- FIM DA MODIFICAÇÃO ---
        }
    }
```

### 4\. Como Receber o Broadcast

Qualquer outro aplicativo (ou componente do sistema) agora pode registrar um `BroadcastReceiver` para "ouvir" esta `Intent`.

**Exemplo (AndroidManifest.xml de outro app):**

```xml
<receiver android:name=".MyPlaybackStateReceiver"
          android:exported="true">
    <intent-filter>
        <action android:name="com.meu.device.ACTION_GENERIC_PLAYBACK_STATE_CHANGED" />
    </intent-filter>
</receiver>
```

**Exemplo (MyPlaybackStateReceiver.java):**

```java
public class MyPlaybackStateReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if ("com.meu.device.ACTION_GENERIC_PLAYBACK_STATE_CHANGED".equals(intent.getAction())) {
            String packageName = intent.getStringExtra("packageName");
            boolean isPlaying = intent.getBooleanExtra("isUserEngaged", false);
            int state = intent.getIntExtra("playbackState", -1);

            // Agora você pode agir com base nessa informação
            // (ex: disparar a sua lógica de auto-rotação)
        }
    }
}
```
