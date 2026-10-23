# Integración Laravel ↔ n8n (workflow 03 - Seguimiento de Trial)

Ajusta los nombres de modelo/columnas (`Suscripcion`, `estado`, `email`, etc.) a los de tu proyecto.

## 1. Variables de entorno (`.env`)

```env
N8N_WEBHOOK_TRIAL=https://TU-N8N.com/webhook/storedimo-nuevo-trial
N8N_TOKEN=genera-un-token-largo-y-aleatorio
```

`config/services.php`:

```php
'n8n' => [
    'webhook_trial' => env('N8N_WEBHOOK_TRIAL'),
    'token'         => env('N8N_TOKEN'),
],
```

## 2. Avisar a n8n cuando se crea una suscripción Trial

En el controlador donde guardas la suscripción (el formulario "Crear Suscripción Empresa"), después de guardar:

```php
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

if ($suscripcion->plan === 'Trial') {
    try {
        Http::timeout(5)
            ->withHeaders(['X-N8N-Token' => config('services.n8n.token')])
            ->post(config('services.n8n.webhook_trial'), [
                'suscripcion_id' => $suscripcion->id,
                'nombre'         => $empresa->nombre_contacto,
                'empresa'        => $empresa->nombre,
                'email'          => $empresa->email,
                'telefono'       => $empresa->celular,
                'plan'           => $suscripcion->plan,
                'fecha_fin'      => $suscripcion->fecha_final,
            ]);
    } catch (\Throwable $e) {
        // Que una falla de n8n nunca bloquee el registro del cliente
        Log::warning('No se pudo notificar a n8n: '.$e->getMessage());
    }
}
```

## 3. Endpoint para que n8n consulte si ya pagó

`routes/api.php`:

```php
use App\Models\Suscripcion;

Route::get('/n8n/suscripciones/{id}/estado', function (Request $request, $id) {
    abort_unless(
        hash_equals((string) config('services.n8n.token'), (string) $request->header('X-N8N-Token')),
        401
    );

    $suscripcion = Suscripcion::findOrFail($id);

    return response()->json([
        // Ajusta esta condición a tu lógica de pago
        'pagado' => $suscripcion->plan !== 'Trial' && $suscripcion->estado === 'activa',
    ]);
});
```

## 4. Credenciales en n8n

Crea una credencial **Header Auth** con:
- Name: `X-N8N-Token`
- Value: el mismo valor de `N8N_TOKEN`

Asígnala al nodo Webhook "Nuevo trial (desde Laravel)" y a los dos nodos "¿Ya pagó?".
