# UserProfileRepository.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `010b4ac392ba930d5f14635afc91262b7e48bd99`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/profile/repository/UserProfileRepository.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 41–99행

<a id="source-L49"></a>

````java
      @Param("active") Boolean active,
      @Param("pushConsent") Boolean pushConsent,
      Pageable pageable);

  /** 특정 상태의 사용자 프로필을 PK로 조회한다. */
  Optional<UserProfile> findByIdAndStatus(Long id, UserProfileStatus status);

  /** 활성 사용자 프로필을 PK로 조회하면서 프로필 상태 변경을 직렬화한다. */
  @Lock(LockModeType.PESSIMISTIC_WRITE)
  @Query(
      """
            select userProfile
            from UserProfile userProfile
            where userProfile.id = :id
              and userProfile.status = com.landit.landitbe.feature.profile.domain.UserProfileStatus.ACTIVE
      """)
  Optional<UserProfile> findActiveByIdForUpdate(@Param("id") Long id);

  /**
   * 활성 사용자들을 ID 순서로 잠가 일괄 작업과 탈퇴를 직렬화한다.
   *
   * @param ids 조회할 사용자 ID 목록
   * @return ID 오름차순으로 잠근 활성 프로필 목록
   */
  @Lock(LockModeType.PESSIMISTIC_WRITE)
  @Query(
      """
      select profile from UserProfile profile
      where profile.id in :ids
        and profile.status = com.landit.landitbe.feature.profile.domain.UserProfileStatus.ACTIVE
      order by profile.id
      """)
  List<UserProfile> findActiveByIdsForUpdate(@Param("ids") List<Long> ids);

  /**
   * 상태와 무관하게 사용자 프로필을 PK로 조회하면서 구독 상태 변경을 직렬화한다.
   *
   * <p>결제 제공자 웹훅은 탈퇴한 사용자의 환불·만료 이벤트도 반영해야 하므로 활성 조건을 두지 않는다.
   *
   * @param id 사용자 프로필 ID
   * @return 쓰기 잠금으로 조회한 사용자 프로필. 없으면 빈 값
   */
  @Lock(LockModeType.PESSIMISTIC_WRITE)
  @Query("select userProfile from UserProfile userProfile where userProfile.id = :id")
  Optional<UserProfile> findByIdForUpdate(@Param("id") Long id);

  /** 특정 상태의 사용자 프로필 존재 여부를 PK로 확인한다. */
  boolean existsByIdAndStatus(Long id, UserProfileStatus status);

  /**
   * 특정 상태와 역할을 가진 사용자 프로필 존재 여부를 확인한다.
   *
   * @param id 사용자 프로필 ID
   * @param status 확인할 사용자 프로필 상태
   * @param role 확인할 사용자 역할
   * @return 해당 상태와 역할을 가진 프로필이 있으면 {@code true}
   */
  boolean existsByIdAndStatusAndRole(Long id, UserProfileStatus status, UserRole role);
}
````
