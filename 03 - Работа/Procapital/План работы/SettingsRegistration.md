#Work/Procapital 
в SettingsRegistration замени
```javascript
const passedAndNotOpened = Object.entries(statuses).find(
  ([, status]) => status.profile_status === 'verification_passed' && !status.is_opened
);

```