
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
